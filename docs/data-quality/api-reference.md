# DQ API Reference

All DQ endpoints live under `/api/v1/dq`. Authentication uses the same OAuth2 bearer token as the rest of the MDM Core API. Scopes follow the existing RBAC model.

| Scope | Required For |
|---|---|
| `mdm.read` | GET endpoints (scores, history, reports, profiles) |
| `mdm.admin` | POST endpoints (manual evaluation, profiling triggers) |

---

## Score Endpoints

### Get DQ Score for an Object

```
GET /api/v1/dq/{objectType}/{id}/score
```

Returns the current composite score and per-dimension breakdown for a single master data object. The response reads from the latest `DqScoreRecord` in `dq_score_history` — it does not re-run the engine.

**Path Parameters**

| Param | Description |
|---|---|
| `objectType` | Object type (e.g. `customer`, `material`, `vendor`) |
| `id` | Master data object ID |

**Response `200 OK`**

```json
{
  "objectId": "64abc123...",
  "objectType": "customer",
  "compositeScore": 0.87,
  "grade": "B+",
  "dimensionScores": {
    "COMPLETENESS": 0.95,
    "VALIDITY": 0.80,
    "CONFORMITY": 1.00,
    "CONSISTENCY": 0.75,
    "UNIQUENESS": 1.00
  },
  "failedRules": [
    {
      "ruleId": "PHONE_FORMAT",
      "dimension": "VALIDITY",
      "fieldPath": "phone",
      "message": "Value '(415) 555-0123' is not in E.164 format",
      "originalValue": "(415) 555-0123",
      "correctedValue": "+14155550123"
    }
  ],
  "scoredAt": "2025-04-09T14:35:22Z",
  "triggeredBy": "UPDATE"
}
```

The `grade` field maps composite score to a letter grade: A (≥0.95), B (≥0.85), C (≥0.70), D (≥0.55), F (<0.55).

**Response `404 Not Found`** — object ID does not exist or has no score history yet.

---

### Get Score History for an Object

```
GET /api/v1/dq/{objectType}/{id}/history
```

Returns paginated `DqScoreRecord` entries for a single object, ordered by `scoredAt` descending. Use this to chart quality improvement over time.

**Query Parameters**

| Param | Default | Description |
|---|---|---|
| `page` | `0` | Zero-based page number |
| `size` | `20` | Records per page (max 100) |

**Response `200 OK`**

```json
{
  "content": [
    {
      "compositeScore": 0.87,
      "dimensionScores": { ... },
      "scoredAt": "2025-04-09T14:35:22Z",
      "triggeredBy": "UPDATE"
    },
    {
      "compositeScore": 0.72,
      "dimensionScores": { ... },
      "scoredAt": "2025-03-15T09:10:05Z",
      "triggeredBy": "CREATE"
    }
  ],
  "totalElements": 8,
  "totalPages": 1,
  "page": 0,
  "size": 20
}
```

---

### Trigger Manual Re-evaluation

```
POST /api/v1/dq/{objectType}/{id}/evaluate
```

Re-runs the full DQ pipeline (standardization → enrichment → rule evaluation → scoring) for a single object. The standardized and enriched object is **saved** back to `master_data_objects`. A new `DqScoreRecord` is written with `triggeredBy: MANUAL`.

**Response `200 OK`** — same shape as the score GET response above.

**Response `404 Not Found`** — object ID does not exist.

!!! note "Use case"
    Manual re-evaluation is useful after updating the rule configuration for an object type — to immediately see the score impact on existing records without waiting for the next update mutation.

---

## Report Endpoints

### Aggregate DQ Report

```
GET /api/v1/dq/{objectType}/report
```

Returns population-level DQ statistics for all `ACTIVE` records of an object type. The report is computed on-demand from the latest score per object in `dq_score_history`.

**Response `200 OK`**

```json
{
  "objectType": "customer",
  "totalRecords": 42381,
  "averageCompositeScore": 0.79,
  "averageDimensionScores": {
    "COMPLETENESS": 0.88,
    "VALIDITY": 0.72,
    "CONFORMITY": 0.85,
    "CONSISTENCY": 0.74,
    "UNIQUENESS": 0.81
  },
  "scoreDistribution": {
    "0.9-1.0": 8420,
    "0.8-0.9": 14310,
    "0.7-0.8": 11200,
    "0.6-0.7": 5830,
    "below-0.6": 2621
  },
  "topFailingRules": [
    { "ruleId": "PHONE_FORMAT",    "failureRate": 0.58 },
    { "ruleId": "CROSS_FIELD",     "failureRate": 0.26 },
    { "ruleId": "ALLOWED_VALUES",  "failureRate": 0.09 }
  ],
  "reportGeneratedAt": "2025-04-09T14:40:00Z"
}
```

`topFailingRules` lists the rules with the highest failure rate across the population, sorted descending. This directly identifies where standardization effort should be focused.

---

## Profiling Endpoints

### Trigger Bulk Profiling

```
POST /api/v1/dq/{objectType}/profile
```

Starts an asynchronous profiling scan across all `ACTIVE` records of the given object type. Returns immediately with `202 Accepted`.

**Response `202 Accepted`**

```json
{
  "jobId": "profile-customer-20250409-143022",
  "objectType": "customer",
  "status": "RUNNING",
  "startedAt": "2025-04-09T14:30:22Z"
}
```

Use the `GET /profiles` endpoint to check for the completed result.

---

### List Profiling Results

```
GET /api/v1/dq/{objectType}/profiles
```

Returns paginated `DqProfilingResult` documents for an object type, ordered by `profiledAt` descending.

**Query Parameters**

| Param | Default | Description |
|---|---|---|
| `page` | `0` | Zero-based page number |
| `size` | `10` | Records per page |

---

### Get Latest Profile

```
GET /api/v1/dq/{objectType}/profiles/latest
```

Returns the most recent profiling result for an object type. See [Profiling](profiling.md) for the full response schema.

**Response `404 Not Found`** — no profiling has been run for this object type yet.

---

## Error Responses

All endpoints follow the existing `ApiResponse` envelope:

```json
{
  "success": false,
  "message": "Object type 'invoice' is not configured for DQ evaluation",
  "timestamp": "2025-04-09T14:41:00Z"
}
```

| HTTP Status | Meaning |
|---|---|
| `400 Bad Request` | Invalid path parameter or unsupported object type |
| `401 Unauthorized` | Missing or invalid bearer token |
| `403 Forbidden` | Valid token but insufficient scope |
| `404 Not Found` | Object or profile does not exist |
| `202 Accepted` | Async operation started (profiling trigger) |
| `500 Internal Server Error` | Unexpected engine failure — check logs |

---

## OpenAPI / Swagger

The DQ endpoints are registered in the existing Springdoc OpenAPI spec at `/swagger-ui.html`. All endpoints appear under the `data-quality` tag.
