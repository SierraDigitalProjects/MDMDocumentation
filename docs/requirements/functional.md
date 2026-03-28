# Functional Requirements

Requirements are tagged with the management tier they apply to:
`[C]` = Centrally Managed | `[L]` = Locally Managed | `[M]` = MDM Lead System | `[ALL]` = All tiers

---

## FR-01 — Master Data Object Support

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-01.1 | MDM must support all 83 master data objects identified in the inventory | ALL | Must Have |
| FR-01.2 | MDM must support the full attribute set for each object as defined in SAP S/4HANA where applicable | C, L | Must Have |
| FR-01.3 | MDM must support custom attributes for non-SAP objects (CRM, EH&S, HR) not defined in SAP | M | Must Have |
| FR-01.4 | MDM must allow attribute schema extension without platform upgrade | ALL | Should Have |
| FR-01.5 | MDM must support object versioning — every change creates a new version, previous versions are retained | ALL | Must Have |

---

## FR-02 — Lead System and Replication

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-02.1 | MDM must receive inbound replication from SAP S/4HANA for all centrally managed objects | C | Must Have |
| FR-02.2 | MDM must receive inbound replication from SAP S/4HANA for all locally managed objects, tagged with site identifier | L | Must Have |
| FR-02.3 | MDM must publish approved records to downstream systems for MDM-lead objects | M | Must Have |
| FR-02.4 | Replication must be near-real-time (< 5 minutes latency) for create/update events on critical objects (Material Master, Business Partner, Equipment) | C, L | Must Have |
| FR-02.5 | Replication must support both full-load (initial migration) and delta-load (ongoing sync) modes | ALL | Must Have |
| FR-02.6 | MDM must maintain a replication status indicator per object showing last sync time and status | ALL | Must Have |

---

## FR-03 — Match and Merge

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-03.1 | MDM must apply deterministic matching before probabilistic matching for all inbound records | ALL | Must Have |
| FR-03.2 | MDM must auto-merge records above a configurable high-confidence threshold | ALL | Must Have |
| FR-03.3 | MDM must route potential duplicates below the high-confidence threshold to a manual review queue | ALL | Must Have |
| FR-03.4 | The Business Partner object must support cross-domain matching (CRM Prospect vs. SAP Customer vs. SAP Vendor) | M, C | Must Have |
| FR-03.5 | MDM must support survivor rules — configuring which source system's attribute value "wins" in a merge | ALL | Must Have |
| FR-03.6 | Merged records must retain links to all source system IDs (cross-reference table) | ALL | Must Have |

---

## FR-04 — Workflow and Approval

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-04.1 | All new record creation must go through a configurable approval workflow before becoming a golden record | ALL | Must Have |
| FR-04.2 | All updates to existing golden records must go through approval workflow | ALL | Must Have |
| FR-04.3 | Approval workflows must be configurable per object type (different approvers for Material Master vs. Customer) | ALL | Must Have |
| FR-04.4 | MDM must support multi-step approval chains (e.g., steward review → lead approval) | ALL | Should Have |
| FR-04.5 | Workflow tasks must be accessible via email notification with approve/reject links | ALL | Must Have |
| FR-04.6 | Approval escalation must trigger automatically if a task is not actioned within a configurable time window | ALL | Should Have |
| FR-04.7 | Emergency bypass must be available for urgent records, with mandatory audit justification | ALL | Should Have |

---

## FR-05 — Role-Based Access Control

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-05.1 | MDM must enforce site-scoped write permissions for all locally managed objects | L | Must Have |
| FR-05.2 | Site Data Coordinators must be unable to create or modify records owned by a different site | L | Must Have |
| FR-05.3 | All authenticated users must be able to read all approved golden records regardless of site | ALL | Must Have |
| FR-05.4 | Role assignments must be manageable by MDM Platform Admins without vendor support | ALL | Must Have |
| FR-05.5 | MDM must integrate with the enterprise Identity Provider (IdP) for SSO authentication | ALL | Must Have |
| FR-05.6 | Service accounts for system integrations must have minimal privilege (object-type specific) | ALL | Must Have |

---

## FR-06 — Data Quality

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-06.1 | MDM must enforce configurable validation rules per object type | ALL | Must Have |
| FR-06.2 | Validation errors must block record approval (not just warn) for Error-severity rules | ALL | Must Have |
| FR-06.3 | MDM must produce a data quality score per golden record based on completeness, validity, and uniqueness | ALL | Should Have |
| FR-06.4 | MDM must provide a data quality dashboard showing scores by object type and module | ALL | Should Have |
| FR-06.5 | Reference data (material groups, GL account groups, etc.) must be maintained within MDM and used for validation | ALL | Must Have |

---

## FR-07 — Search and Discovery

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-07.1 | MDM must provide full-text search across all golden records | ALL | Must Have |
| FR-07.2 | Search must support filtering by object type, module, site, management tier, and status | ALL | Must Have |
| FR-07.3 | MDM must expose a REST API for programmatic record lookup by consumers | ALL | Must Have |
| FR-07.4 | API must support search by source system ID (e.g., SAP Material Number, Salesforce Account ID) | ALL | Must Have |

---

## FR-08 — Audit and History

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-08.1 | Every change to a golden record must be logged with user, timestamp, old value, and new value | ALL | Must Have |
| FR-08.2 | Complete approval chain must be stored with each record version | ALL | Must Have |
| FR-08.3 | Audit log must be immutable — no user, including Platform Admin, can delete or modify log entries | ALL | Must Have |
| FR-08.4 | Audit log must be exportable for compliance reporting | ALL | Must Have |

---

## FR-09 — Data Enrichment

Requirements for governing how satellite systems, stewards, and third-party providers enrich master data objects with additional attributes beyond what the lead system provides.

For the solution design, see [Data Enrichment Governance](../architecture/data-enrichment-governance.md).

| ID | Requirement | Tier | Priority |
|---|---|---|---|
| FR-09.1 | Any satellite system must be able to add its own extension attributes to a shared MDM object without modifying the core object schema | ALL | Must Have |
| FR-09.2 | Extension attributes must be isolated per satellite system in a named namespace — one system cannot read or write another system's namespace without explicit scope grant | ALL | Must Have |
| FR-09.3 | A satellite system must register its extension namespace (field names, types, update frequency) before being permitted to write to it | ALL | Must Have |
| FR-09.4 | Extension namespace registration must be approved by the Data Steward Lead | ALL | Must Have |
| FR-09.5 | Each extension field must carry provenance metadata: owning system, last updated timestamp, and source reference | ALL | Must Have |
| FR-09.6 | Data Stewards must be able to manually enrich core attributes with a justification and source reference, subject to peer review approval | ALL | Must Have |
| FR-09.7 | MDM must support third-party data providers as enrichment sources, with provider onboarding governed by the data governance committee | ALL | Should Have |
| FR-09.8 | MDM must support computed / derived fields that are automatically recalculated when source fields change, and are read-only to all external systems | ALL | Should Have |
| FR-09.9 | Consumers must be able to discover available extension namespaces per object type via the Schema Definition API, including field list, owner, and update frequency | ALL | Must Have |
| FR-09.10 | Consumers must be able to register interest in extension fields and receive events only when those specific extension fields change | ALL | Must Have |
| FR-09.11 | PII fields in extension namespaces must require an elevated access scope and data privacy approval before a consumer can subscribe to them | ALL | Must Have |
| FR-09.12 | Enrichment conflicts (manual enrichment contradicts lead system value) must be surfaced as a conflict flag for steward resolution, not silently overwritten | ALL | Must Have |
| FR-09.13 | Stale enrichment fields (not updated within the defined threshold) must be flagged and excluded from consumer events until refreshed | ALL | Should Have |
| FR-09.14 | All enrichment changes must be captured in the immutable audit log with enrichment type, namespace, source system, and source reference | ALL | Must Have |
