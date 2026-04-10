# Data Quality Engine

The Sierra Aura DQ Engine provides automated profiling, standardization, enrichment, and multi-dimensional scoring across all master data domains. It runs as an integrated pipeline within the MDM Core API, triggered on every create, update, and extension mutation.

---

## Why a Dedicated DQ Engine

The original `DataQualityService` computed a single completeness ratio — the fraction of non-null fields. This worked as a bootstrap signal but had significant blind spots:

- A record with all fields populated but an invalid phone number scored 1.0
- A record with a misspelled country code scored the same as a valid one
- There was no mechanism to fix known issues (standardization)
- There was no way to derive missing fields from existing data (enrichment)
- Scores could not be trended over time

The DQ Engine replaces this with a five-dimension framework, a pluggable rule registry, and a full profiling capability — while remaining backward-compatible with all existing callers.

---

## Five-Dimension Scoring Model

Every master data object receives a composite score (0.0–1.0) computed as a weighted average across five dimensions.

| Dimension | Default Weight | What It Measures |
|---|---|---|
| **Completeness** | 30% | Required and expected fields are present and non-blank |
| **Validity** | 25% | Field values match expected formats (regex, E.164 phone, ISO-8601 date) |
| **Conformity** | 20% | Values belong to approved domain sets (country codes, units of measure) |
| **Consistency** | 15% | Cross-field logic holds (e.g. postal code aligns with country) |
| **Uniqueness** | 10% | No active duplicate records share key identity fields |

Dimension weights are configurable per deployment in `application.yml`. They must sum to 1.0.

```
Composite Score = Σ (dimension_score × dimension_weight)
```

---

## Pipeline Architecture

Each mutation (create, update, or extension update) triggers the following pipeline synchronously before the record is saved:

```
Incoming MasterDataObject
         │
         ▼
┌─────────────────────┐
│  1. Standardization │  ← Normalize phone (E.164), email (lowercase),
│                     │    names (title case), dates (ISO-8601)
└────────┬────────────┘
         │  standardized object
         ▼
┌─────────────────────┐
│  2. Enrichment      │  ← Derive fullName, region from postalCode,
│                     │    country display name from code
└────────┬────────────┘
         │  enriched object
         ▼
┌─────────────────────┐
│  3. Rule Evaluation │  ← All configured rules run against the enriched
│                     │    object; each returns a DqRuleResult (0.0–1.0)
└────────┬────────────┘
         │  rule results
         ▼
┌─────────────────────┐
│  4. Score Compute   │  ← Per-dimension averages → weighted composite
└────────┬────────────┘
         │
         ▼
   Save to MongoDB
         │
         ▼  (async, non-blocking)
┌─────────────────────┐
│  5. Score History   │  ← DqScoreRecord written to dq_score_history
│                     │    collection for trending
└─────────────────────┘
```

!!! note "Standardization is synchronous"
    Steps 1 and 2 apply corrections to the object **before** `masterDataRepository.save()` — the standardized, enriched data is what gets persisted. Steps 4 and 5 are fire-and-forget: a DQ pipeline failure never blocks a mutation.

---

## Domain Coverage

The DQ Engine ships with rule configurations for all seven supported object types.

| Object Type | Completeness Rules | Validity Rules | Conformity Rules | Consistency Rules | Uniqueness Rules |
|---|---|---|---|---|---|
| `customer` | firstName, lastName, email, phone | email regex, phone E.164, date format | country ISO, currency code | postalCode ↔ country | email |
| `material` | description, baseUnit | EAN format (8 or 13 digits) | baseUnit (EA/KG/L/M/PC…) | — | — |
| `vendor` | name, country, phone | phone E.164 | country ISO | — | taxId |
| `employee` | firstName, email | email regex | — | — | employeeId |
| `businesspartner` | name, phone | phone E.164 | country ISO | — | taxId, sapBpNumber |
| `crm-prospect` | email | email regex | — | — | — |
| `equipment` | serialNumber, manufacturer | — | — | — | serialNumber |

---

## Data Flow in MdmCoreService

The DQ Engine integrates at three points in `MdmCoreService`:

```
createObject()       → standardize → enrich → save → async score + history
doUpdateObject()     → standardize → enrich → save → async score + history
updateExtension()    → standardize → enrich → save → async score + history
```

The `dataQualityScore` field in `MdmMetadata` continues to hold the formatted composite score (e.g. `"0.87"`) for backward compatibility. Two new fields are added:

- `dimensionScores` — `Map<String, Double>` keyed by dimension name
- `dqLastEvaluatedAt` — ISO-8601 timestamp of the last evaluation

---

## MongoDB Collections

| Collection | Purpose |
|---|---|
| `master_data_objects` | Golden records — receives standardized, enriched data |
| `dq_score_history` | Per-object score snapshots indexed by `{objectId, scoredAt}` |
| `dq_profiles` | Bulk profiling results per object type indexed by `{objectType, profiledAt}` |

---

## Related Sections

- [Rules Framework](rules-framework.md) — rule types, configuration, and how to add custom rules
- [Standardization & Enrichment](standardization-enrichment.md) — normalization logic and derivation templates
- [Profiling](profiling.md) — bulk field analysis and score trending
- [API Reference](api-reference.md) — REST endpoints for scores, reports, and profiling triggers
