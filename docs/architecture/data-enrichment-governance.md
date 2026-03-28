# Data Enrichment Governance

Data enrichment in MDM refers to the process of **adding or improving attributes on a golden record beyond what the lead system provides**. This includes satellite system extensions, third-party data appending, and manual steward enrichment.

Enrichment without governance creates shadow data — ungoverned attributes that other consumers may rely on without knowing their source, quality, or update frequency. This page defines the governance model for all enrichment activities on MDM golden records.

---

## Types of Enrichment

| Type | Description | Example |
|---|---|---|
| **System Extension** | A satellite system adds its own attributes to a shared object | CRM adds `crm_tier`, `account_manager` to a Business Partner |
| **Manual Steward Enrichment** | A Data Steward adds or corrects attributes that no source system provides | Data Team adds `preferred_language` to a Customer record |
| **Third-Party Data Append** | External data provider enriches the record (D&B, Dun & Bradstreet, address validation) | Appending validated postal address, DUNS number, credit score |
| **Computed / Derived Fields** | MDM computes a field from existing attributes | `full_address` derived from street + city + postal code + country |
| **Cross-object Linking** | MDM resolves relationships between object types | Linking a Contact to a Business Partner across CRM and SAP |

---

## Enrichment Governance Model

### Ownership by Enrichment Type

| Enrichment Type | Who Can Write | Approval Required | Scope |
|---|---|---|---|
| System Extension | Satellite system service account (scoped role) | No — self-governed within namespace | Own extension namespace only |
| Manual Steward Enrichment | Central Data Steward | Yes — peer review by second steward | Core attributes or shared extensions |
| Third-Party Data Append | MDM integration service account | Yes — Data Steward Lead approves provider onboarding | Defined fields only |
| Computed / Derived Fields | MDM platform (automated) | No — rule defined by Data Steward Lead | Read-only computed fields |
| Cross-object Linking | Central Data Steward | Yes — both object stewards approve | Link records only |

### Namespace Isolation

Every enrichment is contained within a named namespace in the `extensions` map. No satellite system can write outside its own namespace.

```json
{
  "_id": "MDM-CUST-001",
  "objectType": "Customer",
  "coreAttributes": { "name": "Acme Corp", "country": "DE" },
  "extensions": {
    "CRM": {
      "_meta": { "owner": "CRM_TEAM", "lastUpdated": "2024-06-20", "version": 3 },
      "crm_tier": "Gold",
      "account_manager": "john.doe@sierradock.com"
    },
    "ERP": {
      "_meta": { "owner": "ERP_TEAM", "lastUpdated": "2024-05-01", "version": 1 },
      "payment_terms": "NET30",
      "vendor_class": "A"
    },
    "DUNS": {
      "_meta": { "owner": "THIRD_PARTY_DUNS", "lastUpdated": "2024-06-01", "source": "D&B API" },
      "duns_number": "123456789",
      "credit_rating": "A2"
    }
  }
}
```

Each extension namespace carries a `_meta` block recording ownership, last update timestamp, and data source. This makes the provenance of every enriched field transparent to all consumers.

---

## Enrichment Request Workflow

### System Extension Registration

Before a satellite system can write to an extension namespace, it must register the namespace with MDM:

```
1. Satellite system team submits Extension Registration Request
   (objectType, proposed namespace name, field list, field types, update frequency)
         │
         ▼
2. Data Steward Lead reviews
   ├── Does this duplicate an existing extension? → Reject / redirect to existing namespace
   ├── Are field names consistent with MDM naming conventions? → Request corrections
   └── APPROVED → Namespace created, RBAC scope granted to service account
         │
         ▼
3. XSUAA scope created: mdm.extend.{NAMESPACE}
   (e.g. mdm.extend.CRM — only CRM service account holds this scope)
         │
         ▼
4. Extension namespace visible in Schema Definition Service
   (consumers can query what fields exist per namespace)
```

### Manual Steward Enrichment Workflow

```
1. Data Steward identifies missing or incorrect core attribute
         │
         ▼
2. Steward submits enrichment change in MDM UI
   (field, old value, new value, justification, source reference)
         │
         ▼
3. Second Data Steward reviews
   ├── Source reference valid? (e.g. customer confirmation email, contract)
   ├── Change consistent with lead system data?
   └── APPROVED → Field updated, change record written to audit log
         │
         ▼
4. AEM event published: changedFields = ["coreAttributes.{field}"]
   Interested consumers are notified
```

### Third-Party Data Append Onboarding

```
1. Data Steward Lead proposes third-party provider
   (provider name, data fields, update schedule, cost, quality SLA)
         │
         ▼
2. MDM Architect reviews integration pattern
   (REST API, batch file, or Kafka feed from provider)
         │
         ▼
3. Data governance committee approves provider and field list
         │
         ▼
4. CPI iFlow or integration service built for provider inbound
   Fields mapped to extension namespace: extensions.{PROVIDER_NAME}
         │
         ▼
5. Provider namespace registered in Schema Definition Service
   Quality score tracked separately for provider-sourced fields
```

---

## Enrichment Quality Rules

Enriched fields are subject to the same quality framework as core attributes, with additional rules:

| Rule | Applies To | Description |
|---|---|---|
| **Source required** | All enrichment types | Every extension field must have a traceable source in `_meta.source` |
| **Staleness threshold** | Third-party and computed | Fields older than the defined `maxStalenessHours` are flagged as stale and excluded from consumer events |
| **Conflict detection** | Manual enrichment on core attributes | If a manual enrichment contradicts the lead system value, a conflict flag is raised for steward resolution |
| **Namespace schema validation** | System extensions | Fields must conform to the registered schema for the namespace — unregistered fields are rejected |
| **No PII in unscoped extensions** | All enrichment | PII fields (SSN, bank account, salary) must not be placed in extension namespaces without explicit data privacy approval and field-level access restriction |

---

## Enrichment Visibility for Consumers

Consumers can discover what enrichment is available per object type via the Schema Definition API:

```
GET /api/v1/schema/Customer/extensions

Response:
[
  {
    "namespace": "CRM",
    "owner": "CRM_TEAM",
    "fields": ["crm_tier", "account_manager", "opportunity_count"],
    "updateFrequency": "REALTIME",
    "requiredScope": "mdm.read"
  },
  {
    "namespace": "DUNS",
    "owner": "THIRD_PARTY_DUNS",
    "fields": ["duns_number", "credit_rating", "employee_count"],
    "updateFrequency": "MONTHLY",
    "requiredScope": "mdm.read.finance"   ← restricted scope
  }
]
```

Consumers register field interest in the Consumer Registry including extension fields:

```json
{
  "consumerId": "CREDIT_SERVICE",
  "objectType": "Customer",
  "interestedFields": [
    "coreAttributes.name",
    "coreAttributes.country",
    "extensions.DUNS.credit_rating",
    "extensions.ERP.payment_terms"
  ]
}
```

The Consumer Registry validates that the registering service account holds the required scope for each extension namespace it wants to subscribe to.

---

## Enrichment Audit Trail

Every enrichment write is recorded in the immutable `change_records` collection with full provenance:

```json
{
  "objectId": "MDM-CUST-001",
  "objectType": "Customer",
  "version": 43,
  "eventType": "enriched",
  "changedFields": ["extensions.CRM.crm_tier"],
  "changedBy": "crm-service-account@sierradock.com",
  "changedBySystem": "CRM_SYSTEM",
  "enrichmentNamespace": "CRM",
  "sourceReference": "CRM Account ID: ACC-00123",
  "changedAt": "2024-06-20T14:30:00Z"
}
```

Data stewards and auditors can query the full enrichment history per object, per namespace, or per source system at any time.

---

## Governance Summary

| Principle | Implementation |
|---|---|
| **Every enriched field has an owner** | Extension namespace registration with named owner team |
| **Write access is narrowly scoped** | XSUAA scope per namespace — service accounts cannot cross namespace boundaries |
| **Every change is traceable** | `_meta` block + immutable `change_records` audit log |
| **Consumers know what they're getting** | Schema Definition API exposes provenance and update frequency per namespace |
| **Quality is measured separately** | Extension fields carry staleness flags and conflict flags independent of core attributes |
| **PII is protected** | Extension PII fields require elevated scope and data privacy sign-off |
