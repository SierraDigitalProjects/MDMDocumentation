# Standardization & Enrichment

Standardization and enrichment run **before** `masterDataRepository.save()` — the corrected, derived data is what gets persisted to MongoDB. This distinguishes them from scoring rules, which are informational and run asynchronously.

---

## Standardization

The `DefaultFieldStandardizer` collects `correctedValue` outputs from all validity rules and applies them back to a cloned `coreAttributes` map. The original value is preserved in the corresponding `DqRuleResult` for audit purposes.

### Normalizations Applied Automatically

| Field Type | Input Example | Normalized Output | Rule |
|---|---|---|---|
| Email address | `John.DOE@Example.COM` | `john.doe@example.com` | `REGEX_FORMAT` (lowercase applied on match) |
| Phone number | `(415) 555-0123` | `+14155550123` | `PHONE_FORMAT` |
| Phone number | `+49 30 12345678` | `+493012345678` | `PHONE_FORMAT` |
| Date | `12/31/2024` | `2024-12-31` | `DATE_FORMAT` |
| Date | `31-12-2024` | `2024-12-31` | `DATE_FORMAT` |
| Person name | `john doe` | `John Doe` | Name title-case (built into standardizer) |
| Postal code | `sw1a 1aa` | `SW1A 1AA` | Postal code uppercase (built into standardizer) |

### How It Works

```
Validity rules run → each returns DqRuleResult
                          │
                          ▼
          correctedValue != null ?
               │              │
              YES             NO
               │
               ▼
  DefaultFieldStandardizer
  clones coreAttributes map
  applies correctedValue at fieldPath
               │
               ▼
  Returns standardized MasterDataObject
  (caller passes this to save())
```

The standardizer never modifies the passed object in place — it returns a new object with a cloned attributes map. This preserves the pre-standardization state for rule result comparison.

### Phone Number Normalization

Phone normalization handles:

- Leading zeros (European style): `030 12345678` → `+493012345678`
- Parenthesized area codes: `(415) 555-0123` → `+14155550123`
- Dashes and spaces: `415-555-0123` → `+14155550123`
- Already-normalized E.164: `+14155550123` → `+14155550123` (no-op)

The `defaultCountryCode` param is applied only when the value lacks a country prefix. Configure it per object type:

```yaml
- rule-id: PHONE_FORMAT
  dimension: VALIDITY
  field-path: phone
  params:
    defaultCountryCode: "+49"   # Germany default for vendor records
```

---

## Enrichment

The `DerivedFieldEnricher` appends computed fields to `coreAttributes`. Enrichment runs after standardization, so derivations use already-normalized values as input.

### Template-Based Derivation

Derives a new field by concatenating or formatting existing fields using a Mustache-style template.

```yaml
enrichment:
  - target-field: fullName
    source-fields: [firstName, lastName]
    template: "{{firstName}} {{lastName}}"
```

If any source field is null or blank, the derivation is skipped — `fullName` is not written with a partial value. The `target-field` is only set when all `source-fields` are present.

### Lookup-Based Derivation

Derives a new field by looking up a value in a reference collection based on an existing field.

```yaml
enrichment:
  - target-field: region
    source-fields: [postalCode, country]
    lookup-collection: postal_code_regions
    lookup-key-field: postalCode
    lookup-value-field: region
```

The lookup result is cached per unique key using Caffeine (in-process, TTL 10 minutes) to avoid a MongoDB round-trip for every record in a bulk load.

### Supported Derivations by Object Type

| Object Type | Target Field | Source | Method |
|---|---|---|---|
| `customer` | `fullName` | `firstName` + `lastName` | Template |
| `customer` | `region` | `postalCode` + `country` | Lookup (`postal_code_regions`) |
| `businesspartner` | `fullName` | `firstName` + `lastName` | Template |
| `employee` | `fullName` | `firstName` + `lastName` | Template |

### Adding a Custom Derivation

Add an entry to the `enrichment` block under the target object type in `application.yml`. No code change is required for template or lookup derivations.

For complex derivations (e.g. computing a hash, calling an external service), implement the `FieldEnricher` interface:

```java
// com.sierra.mdm.dq.enricher.FieldEnricher
public interface FieldEnricher {
    MasterDataObject enrich(MasterDataObject obj, List<EnrichmentConfig> config);
}
```

---

## Enrichment Config Schema

```yaml
enrichment:
  - target-field: <string>           # coreAttributes key to write the derived value to
    source-fields: [<string>, ...]   # coreAttributes keys used as input
    template: "<string>"             # optional: Mustache-style template
    lookup-collection: <string>      # optional: MongoDB collection for lookup
    lookup-key-field: <string>       # optional: field in the lookup collection to match on
    lookup-value-field: <string>     # optional: field in the lookup collection to return
```

Either `template` or `lookup-collection` must be provided (not both). If `template` is provided, `lookup-*` fields are ignored.

---

## Audit Trail

Every standardization correction is recorded in the `DqRuleResult`:

```json
{
  "ruleId": "PHONE_FORMAT",
  "fieldPath": "phone",
  "passed": true,
  "score": 1.0,
  "originalValue": "(415) 555-0123",
  "correctedValue": "+14155550123"
}
```

These results are stored in the `DqScoreRecord` in `dq_score_history`, giving a full audit of every normalization applied to a record over its lifetime.

!!! note "Extension attributes"
    Standardization and enrichment currently apply only to `coreAttributes`. Extension namespace attributes (`extensions.*`) are read-only from the DQ engine's perspective — they are written by source systems and not modified by the pipeline.
