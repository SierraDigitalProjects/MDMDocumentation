# Centrally Managed in SAP — Data Team

These objects are maintained in SAP S/4HANA by the central Data Team. SAP is the **system of record**. MDM should consume these objects from SAP, not override them.

!!! success "MDM Role: Consumer"
    MDM subscribes to SAP as the source of truth. Integration pattern: SAP → CPI → MDM (inbound replication).

---

## Finance (FI/CO)

| Object | Description | Notes |
|---|---|---|
| Activity Rates | Cost rates for activity types | CO module |
| Activity Types | Categories of work performed in cost centers | CO module |
| Cost Center | Organizational unit for cost allocation | Core FI/CO object |
| Cost Elements | P&L account classifications for cost tracking | Links to GL Accounts |
| Cost Element Groups | Groupings of cost elements for reporting | |
| GL Accounts | General Ledger chart of accounts | Foundation for all Finance objects |

**Integration consideration:** GL Accounts and Cost Center are foundational — all other Finance objects depend on them. Replication order matters.

---

## SCM / Purchasing (MM)

| Object | Description | Notes |
|---|---|---|
| Material Classifications | Classification of materials for search/reporting | Shared with PM |
| Outline Agreements | Long-term purchasing contracts | |
| Material Master — MRO | Maintenance, Repair & Operations materials | Separate from Raw/Finished |
| Purchase Info Record | Vendor-material price/delivery info | |
| Source List | Approved vendor sources per material | |
| Service Master | Master data for procurement of services | |
| Material Master — Raw & Finished | Production input and output materials | Key MDM object |
| Business Partners Master | Vendor/supplier master | Links to SD BP Master |
| Terminal Agreement | Terminal-specific supply agreements | |
| Standard Purchasing Text | Standard text blocks for POs | |
| Taxes | Tax condition master for purchasing | |

**Integration consideration:** Material Master (both MRO and Raw/Finished) is the single highest-volume and highest-complexity object. Plan for delta replication and deduplication logic.

---

## Sales & Distribution (SD)

| Object | Description | Notes |
|---|---|---|
| BP Master | Business Partner master (customer/vendor unified) | S/4HANA unified BP model |
| Customer Classification | Customer segmentation attributes | |
| Customer Hierarchy | Customer reporting hierarchy | Important for rebate/pricing |
| Standard Text | Standard text blocks for sales documents | |
| Pricing Condition Records | Sales pricing master | High volume, site-variant |
| Freight Master Data | Freight lane and rate master | |

**Integration consideration:** BP Master in SD and Business Partners Master in SCM both derive from the unified S/4HANA Business Partner model — treat as a single object with dual-module relevance.

---

## Plant Maintenance (PM)

| Object | Description | Notes |
|---|---|---|
| Characteristics | Classification system attributes | Shared with PP/QM |
| Classes | Classification system class definitions | |
| Equipment | Individual physical asset records | Core PM object |
| Functional Location | Hierarchical plant/location structure | Parent of Equipment |
| Engineering Change Numbers | ECN/ECO records for controlled changes | Cross-functional |

---

## Project Systems (PS)

| Object | Description | Notes |
|---|---|---|
| Network Header | Project network header records | |
| WBS Elements | Work Breakdown Structure elements | Core PS object |
| Profit Center | Profit center master | Also used by FI/CO |

---

## Other (Cross-Module)

| Object | Description | Notes |
|---|---|---|
| Asset Master | Fixed asset master records | FI-AA module |
| Storage Bins | Warehouse storage location master | WM module |
