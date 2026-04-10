# Profiling & Score Trending

The DQ Engine provides two types of historical analysis: **bulk profiling** (population-level field statistics for a whole object type) and **score trending** (per-object DQ score history over time).

---

## Bulk Profiling

### What Profiling Computes

The `DqProfiler` scans all `ACTIVE` records of a given object type and computes per-field statistics. Results are stored in the `dq_profiles` MongoDB collection.

For each field found in `coreAttributes` across the population, the profiler computes:

| Statistic | Description |
|---|---|
| `nullRate` | Fraction of records where the field is null (0.0–1.0) |
| `blankRate` | Fraction where the field is present but empty string |
| `distinctCount` | Number of unique values |
| `topValues` | Top 10 values by frequency (`Map<String, Long>`) |
| `formatDistribution` | Fraction of values matching each of a set of format regex buckets |

### Format Buckets

The profiler classifies string field values into format buckets to reveal data quality patterns without exposing raw PII:

| Bucket | Pattern | Example match |
|---|---|---|
| `email` | `.*@.*\..*` | `user@example.com` |
| `e164_phone` | `^\+[1-9]\d{6,14}$` | `+14155550123` |
| `local_phone` | `^[\d\s\-().+]{7,20}$` | `(415) 555-0123` |
| `iso_date` | `^\d{4}-\d{2}-\d{2}$` | `2024-12-31` |
| `us_date` | `^\d{2}/\d{2}/\d{4}$` | `12/31/2024` |
| `iso3166_country` | `^[A-Z]{2}$` | `US`, `DE` |
| `numeric` | `^\d+(\.\d+)?$` | `12345`, `3.14` |
| `alphanumeric` | `^[A-Za-z0-9]+$` | `ABC123` |
| `other` | (catch-all) | anything else |

This tells a data steward, for example, that 70% of `phone` values are in `local_phone` format and need E.164 standardization before they can be reliably matched.

### Profiling Result Document

```json
{
  "id": "6789abc...",
  "objectType": "customer",
  "profiledAt": "2025-04-09T14:30:00Z",
  "recordCount": 42381,
  "fieldStats": {
    "email": {
      "nullRate": 0.02,
      "blankRate": 0.00,
      "distinctCount": 41900,
      "topValues": {},
      "formatDistribution": {
        "email": 0.95,
        "other": 0.05
      }
    },
    "phone": {
      "nullRate": 0.11,
      "blankRate": 0.03,
      "distinctCount": 38400,
      "topValues": {},
      "formatDistribution": {
        "e164_phone": 0.42,
        "local_phone": 0.53,
        "other": 0.05
      }
    },
    "country": {
      "nullRate": 0.00,
      "blankRate": 0.00,
      "distinctCount": 12,
      "topValues": {
        "US": 28400,
        "DE": 6300,
        "GB": 3100,
        "FR": 2100,
        "other": 2481
      },
      "formatDistribution": {
        "iso3166_country": 0.91,
        "other": 0.09
      }
    }
  }
}
```

### Running a Profile

Profiling is triggered via the REST API (see [API Reference](api-reference.md)) or can be scheduled:

```
POST /api/v1/dq/customer/profile
```

The scan runs asynchronously — the endpoint returns immediately with a `202 Accepted` and a `jobId`. Progress can be polled at `GET /api/v1/dq/{objectType}/profiles`.

!!! warning "Profile on large populations"
    For object types with millions of records, profiling uses MongoDB cursor streaming (not `findAll()`) to avoid memory exhaustion. Allow sufficient time for large scans. The result document can be large for object types with many distinct field names.

---

## Score Trending

### DqScoreRecord

Every time an object is evaluated (on create, update, or manual trigger), a `DqScoreRecord` is written asynchronously to the `dq_score_history` collection:

```json
{
  "id": "abc123...",
  "objectId": "master_data_objects/_id",
  "objectType": "customer",
  "compositeScore": 0.87,
  "dimensionScores": {
    "COMPLETENESS": 0.95,
    "VALIDITY": 0.80,
    "CONFORMITY": 1.00,
    "CONSISTENCY": 0.75,
    "UNIQUENESS": 1.00
  },
  "ruleResults": [ ... ],
  "scoredAt": "2025-04-09T14:35:22Z",
  "triggeredBy": "UPDATE"
}
```

`triggeredBy` is one of `CREATE`, `UPDATE`, `EXTENSION_UPDATE`, or `MANUAL`.

### Trending Queries

The `dq_score_history` collection has compound indexes on:

- `{ objectId: 1, scoredAt: -1 }` — for per-object history
- `{ objectType: 1, scoredAt: -1 }` — for population-level trend reports

Example: retrieve the last 30 scores for a specific customer to chart quality improvement over time:

```
GET /api/v1/dq/customer/{id}/history?page=0&size=30
```

### Aggregate DQ Report

A summary report for all records of an object type can be retrieved at any time:

```
GET /api/v1/dq/customer/report
```

Response structure:

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
  "reportGeneratedAt": "2025-04-09T14:40:00Z"
}
```

---

## Retention Policy

The `dq_score_history` and `dq_profiles` collections should be managed with a TTL index to prevent unbounded growth. Recommended defaults:

| Collection | Suggested TTL | Rationale |
|---|---|---|
| `dq_score_history` | 365 days | Sufficient for annual trend analysis |
| `dq_profiles` | 90 days | Profile snapshots older than a quarter are rarely consulted |

TTL indexes can be created via `mongosh` or added to the application's `@PostConstruct` index setup:

```javascript
db.dq_score_history.createIndex(
  { "scoredAt": 1 },
  { expireAfterSeconds: 31536000 }
)
```
