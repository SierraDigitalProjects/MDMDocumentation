# RBAC and Data Governance

## Governance Model

The three management tiers require three distinct governance models within MDM.

| Tier | Who Creates | Who Approves | Who Can Edit | Who Can Read |
|---|---|---|---|---|
| Centrally Managed | Data Team | Data Team Lead | Data Team only | All authenticated users |
| Locally Managed | Site Operations | Site MDM Admin | Own site only | All authenticated users |
| Not in SAP (MDM Lead) | MDM steward or source system | Domain steward | Domain team | All authenticated users |

---

## Role Definitions

### Platform Roles

| Role | Scope | Permissions |
|---|---|---|
| **MDM Platform Admin** | Global | Full access; configure rules, roles, workflows |
| **Data Steward — Central** | All objects | Create, read, update, approve for centrally managed objects |
| **Data Steward — Domain** | Domain-specific (CRM, EH&S, etc.) | Create, read, update, approve within domain |
| **Site Data Coordinator** | Site-specific | Create, read, update locally managed objects for own site only |
| **Read-Only Consumer** | Global | Read all approved golden records; no write access |
| **Integration Service Account** | Object-specific | System-to-system API calls; no human login |

### Business Roles (mapped to Platform Roles)

| Business User Type | Mapped Platform Role |
|---|---|
| Central Data Team member | Data Steward — Central |
| Site operations lead | Site Data Coordinator |
| CRM / Sales admin | Data Steward — Domain (CRM) |
| EH&S manager | Data Steward — Domain (EH&S) |
| HR data analyst | Data Steward — Domain (HR) |
| SAP functional users | Read-Only Consumer |
| CPI integration user | Integration Service Account |

---

## Site-Scoped Access Control

For locally managed objects, MDM must enforce **site isolation** at the data level, not just the UI level.

### Requirements

- Every locally managed record must carry a `site_id` attribute
- Write operations must validate `user.site_id == record.site_id`
- Cross-site reads are permitted for all authenticated roles
- Site Data Coordinators cannot promote a local record to a global/shared record — that requires Data Steward approval
- Site-level objects must be tagged in MDM UI to indicate the owning site

### Site Hierarchy

```
SierraDock ND (Enterprise)
├── Site A
│   └── Site Data Coordinator A
├── Site B
│   └── Site Data Coordinator B
└── Site N...
```

---

## Approval Workflows

### Centrally Managed Objects

```
Data Steward submits change request
    │
    ▼
Automated validation rules run
    │
    ├── PASS → Data Steward Lead review
    │               │
    │               ├── Approved → Publish to MDM + Trigger SAP replication
    │               └── Rejected → Return to steward with comments
    │
    └── FAIL → Return to steward with validation errors
```

### Locally Managed Objects (Site-Scoped)

```
Site Data Coordinator submits new/changed record
    │
    ▼
Automated validation rules run
    │
    ├── PASS → Site MDM Admin review
    │               │
    │               ├── Approved → Publish to MDM (site-tagged)
    │               └── Rejected → Return to coordinator
    │
    └── FAIL → Return to coordinator with validation errors
```

### MDM Lead System Objects (Not in SAP)

```
Domain Steward or source system submits record
    │
    ▼
Match & Merge engine evaluates
    │
    ├── No duplicate → Domain Steward review
    │                       │
    │                       ├── Approved → Golden record created/updated
    │                       │               └── Downstream systems notified
    │                       └── Rejected → Archived with reason
    │
    └── Possible duplicate found → Manual merge review required
```

---

## Data Quality Rules

### Universal Rules (All Objects)

| Rule | Description | Severity |
|---|---|---|
| Required fields populated | All mandatory attributes must have values | Error — blocks approval |
| No duplicate golden record | Match score below threshold required | Error — triggers merge review |
| Valid reference data | Codes must exist in approved reference tables | Error — blocks approval |
| Audit trail present | All changes must have user, timestamp, reason | Error — system enforced |

### Object-Specific Rules (Examples)

| Object | Rule | Severity |
|---|---|---|
| Business Partner | Tax ID format valid per country | Warning |
| Material Master | Material group must exist in classification | Error |
| Equipment | Must have a parent Functional Location | Error |
| Hazardous Materials | CAS number must follow format | Warning |
| GL Account | Account must exist in active chart of accounts | Error |
| Pricing Condition Record | Valid-from date must not be in the past | Warning |

---

## Audit and Compliance

All MDM operations must produce a complete audit trail suitable for compliance reporting.

### Minimum Audit Record

```json
{
  "event_id": "uuid",
  "timestamp": "ISO8601",
  "object_type": "MaterialMaster",
  "object_id": "MDM-12345",
  "action": "UPDATE",
  "changed_by": "user@sierradock.com",
  "changed_fields": [
    { "field": "description", "old_value": "...", "new_value": "..." }
  ],
  "approval_chain": [
    { "approver": "steward@sierradock.com", "approved_at": "ISO8601" }
  ],
  "source_system": "SAP_S4HANA",
  "integration_message_id": "CPI-MSG-789"
}
```

### Retention Requirements

| Data Type | Minimum Retention |
|---|---|
| Golden record change history | 7 years |
| Workflow approval records | 7 years |
| Integration message logs | 2 years |
| Failed/error message logs | 1 year |
| User access logs | 3 years |
