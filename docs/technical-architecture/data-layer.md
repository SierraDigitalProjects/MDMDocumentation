# Data Layer — MongoDB, Redis, and Change Tracking

## MongoDB — Primary Store

MongoDB is the correct choice for MDM because:

- **Dynamic schema** — custom-defined object models evolve without migrations
- **Change Streams** — built-in CDC for delta detection per collection
- **JSON Schema validation** — enforce field constraints per object type definition
- **Aggregation pipeline** — efficient field-level diff computation
- **Sharding** — horizontal scale for high object count NFR

### Master Data Object Document Structure

```json
{
  "_id": "MDM-CUST-001",
  "objectType": "Customer",
  "version": 42,
  "coreAttributes": {
    "businessPartnerId": "1000042",
    "name": "Acme Corp",
    "country": "DE",
    "city": "Berlin",
    "postalCode": "10115"
  },
  "extensions": {
    "CRM_SYSTEM": {
      "crm_tier": "Gold",
      "account_mgr": "john.doe@sierradock.com"
    },
    "ERP_SYSTEM": {
      "vendor_class": "A",
      "payment_terms": "NET30"
    }
  },
  "metadata": {
    "leadSystem": "S4HANA",
    "createdBy": "data.team@sierradock.com",
    "createdAt": "2024-01-15T08:00:00Z",
    "updatedBy": "data.team@sierradock.com",
    "updatedAt": "2024-06-20T14:30:00Z"
  },
  "lastChangedFields": [
    "coreAttributes.country",
    "extensions.CRM_SYSTEM.crm_tier"
  ]
}
```

### Extension Attribute Pattern

Extensions allow any satellite system to add fields to a master data object **without changing the core schema**. RBAC controls which system can write to which extension namespace.

```
coreAttributes       → Lead system fields, governed by Data Team
extensions.CRM       → CRM team fields, governed by CRM team RBAC role
extensions.ERP       → ERP team fields, governed by ERP team RBAC role
extensions.SITE_A    → Site-specific fields, governed by Site A coordinator
```

### Recommended Indexes

```javascript
// Compound index for type + updated queries (delta load)
db.master_data_objects.createIndex(
  { objectType: 1, "metadata.updatedAt": -1 }
)

// Index for lead system queries
db.master_data_objects.createIndex(
  { objectType: 1, "metadata.leadSystem": 1 }
)

// Partial index for site-scoped objects
db.master_data_objects.createIndex(
  { objectType: 1, "metadata.siteId": 1 },
  { partialFilterExpression: { "metadata.siteId": { $exists: true } } }
)
```

### Change Records Collection (Audit Log)

```json
{
  "_id": "uuid",
  "objectId": "MDM-CUST-001",
  "objectType": "Customer",
  "version": 42,
  "eventType": "changed",
  "changedFields": [
    "coreAttributes.country",
    "extensions.CRM_SYSTEM.crm_tier"
  ],
  "changedBy": "steward@sierradock.com",
  "changedAt": "2024-06-20T14:30:00Z"
}
```

TTL index on `changedAt` for automatic retention enforcement:

```javascript
db.change_records.createIndex(
  { changedAt: 1 },
  { expireAfterSeconds: 220752000 }  // 7 years
)
```

---

## Field-Level Change Tracking

Change tracking operates at two layers.

### Layer 1 — Application-Level Diff (Primary)

Computed in `FieldChangeTracker` service on every write. Returns dot-notation paths of changed fields.

```
Previous: { coreAttributes: { name: "Acme", country: "US" } }
Current:  { coreAttributes: { name: "Acme", country: "DE" } }

Result:   ["coreAttributes.country"]
```

Extension diffs follow the same pattern:

```
extensions.CRM_SYSTEM.crm_tier changed: "Silver" → "Gold"
Result: ["extensions.CRM_SYSTEM.crm_tier"]
```

### Layer 2 — MongoDB Change Streams (Safety Net)

Catches any writes that bypass the service layer (direct DB writes, migrations, data fixes).

```
MongoDB Change Stream → MongoChangeStreamListener → AEM publish
(fallback CDC — fires only for direct DB writes, not normal API writes)
```

---

## Redis — Read-Through Cache

Redis Stack (RedisJSON + RediSearch) provides:

- **Read-through caching** — serves reads without hitting MongoDB
- **RedisJSON** — store and query JSON documents natively
- **RediSearch** — full-text search across cached objects
- **Cache invalidation** — event-driven via AEM/Kafka consumer on write

### Cache Key Pattern

```
mdm:{objectType}:{objectId}
mdm:customer:MDM-CUST-001
mdm:material:MDM-MAT-005523
```

### Cache TTL Strategy

| Object Type | TTL | Rationale |
|---|---|---|
| Business Partner | 5 min | Moderate change frequency |
| Material Master | 15 min | Changes less frequently |
| Pricing Condition | 1 min | High change frequency, price-sensitive |
| Equipment | 30 min | Rarely changes |
| Reference data | 24 hours | Very stable |

### Read-Through Flow

```
Consumer calls GET /api/v1/objects/customer/MDM-CUST-001
    │
    ▼ Check Redis: mdm:customer:MDM-CUST-001
    │
    ├── HIT  → return cached record (< 5ms)
    │
    └── MISS → fetch from MongoDB
                   │
                   ▼ populate Redis with TTL
                   │
                   ▼ return record
```

### Cache Invalidation

```
MDM Core API writes object
    │
    ├── Saves to MongoDB
    ├── Publishes event to AEM (mdm.cache.invalidation.queue)
    │
    ▼
Cache Invalidation Consumer (internal Kyma service)
    │
    ▼ DEL mdm:customer:MDM-CUST-001
```

---

## Initial Load and Delta Strategy

### Initial Load Approaches

| Approach | When to Use |
|---|---|
| REST Bulk API (chunked, paginated) | Lead system has API capability; preferred for < 10M records |
| Kafka Connect Source Connector (JDBC/File) | Direct DB access from source; high volume |
| Apache NiFi (self-managed on K8s) | Complex transformations, multiple source formats |
| Spring Batch Job (MDM-controlled) | ETL from provided file dumps (CSV / JSON / Parquet) |

Initial load steps:

1. Schema / object type definition created first in MDM
2. Bulk load via chosen mechanism with `suppressEvents=true` flag
3. No fan-out events published during initial load
4. Post-load: publish `INITIAL_LOAD` snapshot event per object so consumers can bootstrap

### Delta / Ongoing Change Mechanisms

| Mechanism | Trigger | Pattern |
|---|---|---|
| Real-time CDC | Any write to MDM | Change Streams → AEM (sub-second) |
| Scheduled delta | Cron per object type | Query `updatedAt > lastRun` → batch publish |
| Lead system push | Inbound API / CPI iFlow | Synchronous write + async AEM publish |
| Watermark-based | Scheduled batch | High-water mark timestamp persisted per object type |

!!! tip "Best Practice"
    Implement **both** real-time CDC (Change Streams) and scheduled delta. Real-time serves REALTIME consumers; scheduled acts as a guaranteed catch-up mechanism and serves batch-only consumers. Never rely on a single delivery mechanism for master data.
