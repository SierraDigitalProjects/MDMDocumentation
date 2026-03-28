# Locally Managed in SAP — Site Operations

These objects are maintained in SAP S/4HANA by individual site operations teams. SAP is the **system of record**, but authority is distributed to sites rather than centralized.

!!! info "MDM Role: Aggregator / Visibility"
    MDM provides cross-site reporting, governance, and consistency checks — without overriding local site authority. Integration: SAP → CPI → MDM (inbound replication, site-tagged).

!!! warning "RBAC Implication"
    MDM must enforce **site-scoped write permissions**. Site A operators must not be able to create or modify objects owned by Site B. This is the primary governance challenge for this tier.

---

## SCM / Purchasing (MM)

| Object | Description | Notes |
|---|---|---|
| Purchasing Condition Record | Site-specific purchase pricing conditions | High site-variance expected |
| Purchasing Output Record | Output/message determination for POs | |

**Why local?** Condition records vary by site based on local vendor negotiations. Centralizing these would remove site flexibility.

---

## Plant Maintenance (PM)

| Object | Description | Notes |
|---|---|---|
| Bill of Materials | Equipment/asset BOMs | Maintained per site |
| Catalogs | PM catalog codes (damage, cause, activity) | Site-specific code sets |
| Maintenance Plan | Preventive maintenance scheduling | Site-specific schedules |
| Maintenance Item | Individual items within maintenance plans | |
| Task Lists | Standard maintenance task procedures | May be shared across sites |
| Standard Text | PM-specific standard text blocks | |
| Work Center | Physical work locations for maintenance | Per site |
| Planner Group | Maintenance planner group assignments | Per site |

**Why local?** Maintenance schedules, work centers, and task lists are driven by site-specific equipment configurations and local regulatory requirements.

---

## PP / Quality Management (QM)

| Object | Description | Notes |
|---|---|---|
| Bill of Materials | Production BOMs | Site-specific production configurations |
| Catalogs | QM inspection code catalogs | |
| Inspection Characteristics | Quality characteristics to be inspected | |
| Inspection Methods | How inspections are performed | |
| Inspection Plan | Full inspection plan per material/operation | |
| Material Consumption | Material usage records in production | |
| Material Classifications | QM-specific material classifications | |
| Work Centers | Production work centers | Per site |
| Sampling Procedures | Statistical sampling rules | |
| Routings | Production routing / operations sequence | Per site |
| Reference Operation Set | Reusable operation groups for routings | |

**Why local?** All PP/QM objects in scope are locally managed — this module has zero centrally managed objects. This suggests quality and production configuration is entirely site-driven at ND.

---

## Project Systems (PS)

| Object | Description | Notes |
|---|---|---|
| Network Activities | Individual activities within project networks | |

---

## Cross-Tier Objects (Shared Naming)

The following object names appear in **both** Centrally Managed and Locally Managed tiers, under different modules:

| Object Name | Central Module | Local Module | Resolution |
|---|---|---|---|
| Bill of Materials | — | PM, PP/QM | Treat as separate objects by module |
| Catalogs | — | PM, PP/QM | Treat as separate objects by module |
| Material Classifications | SCM (Central) | PP/QM (Local) | Same object type, different governance scope — MDM must tag by owning module |
| Work Centers | — | PP/QM | — |
