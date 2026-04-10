# Rules Framework

The DQ Rules Framework is a pluggable, config-driven system. Every rule is a Spring bean that implements the `DqRule` interface. Adding a new rule requires only creating a class — no factory registration, no switch statement, no modification to the engine.

---

## Core Interfaces

### `DqRule`

```java
// com.sierra.mdm.dq.rule.DqRule
public interface DqRule {
    String getRuleId();              // e.g. "PHONE_FORMAT"
    DqDimension getDimension();      // which of the five dimensions this contributes to
    DqRuleResult evaluate(MasterDataObject obj, DqRuleConfig config);
}
```

All rules are annotated `@Component` and collected automatically by `DqRuleRegistry` via Spring's `List<DqRule>` injection.

### `DqRuleResult`

```java
// com.sierra.mdm.dq.rule.DqRuleResult
public class DqRuleResult {
    String ruleId;
    DqDimension dimension;
    String fieldPath;
    boolean passed;
    double score;           // 0.0 (fail) to 1.0 (pass)
    String message;
    Object originalValue;
    Object correctedValue;  // non-null when the rule can suggest a fix (standardization rules)
}
```

### `DqRuleConfig`

```java
// com.sierra.mdm.dq.rule.DqRuleConfig
public class DqRuleConfig {
    String ruleId;
    DqDimension dimension;
    String fieldPath;
    Map<String, String> params;   // rule-specific parameters
}
```

---

## Built-in Rules

### Completeness

#### `FIELD_PRESENT`
**Class:** `com.sierra.mdm.dq.rule.completeness.FieldPresentRule`

Checks that a field in `coreAttributes` (or a dot-notation path into `extensions`) is non-null and non-blank. Returns score `1.0` if present, `0.0` if absent.

```yaml
- rule-id: FIELD_PRESENT
  dimension: COMPLETENESS
  field-path: email
```

---

### Validity

#### `REGEX_FORMAT`
**Class:** `com.sierra.mdm.dq.rule.validity.RegexFormatRule`

Validates a field value against a regular expression. Returns `1.0` if the value matches, `0.0` otherwise. If the field is absent the rule is skipped (returns `1.0`) — use alongside `FIELD_PRESENT` to enforce both presence and format.

| Param | Required | Description |
|---|---|---|
| `pattern` | Yes | Java regex pattern |

```yaml
- rule-id: REGEX_FORMAT
  dimension: VALIDITY
  field-path: email
  params:
    pattern: "^[\\w.+-]+@[\\w-]+\\.[a-zA-Z]{2,}$"
```

#### `PHONE_FORMAT`
**Class:** `com.sierra.mdm.dq.rule.validity.PhoneFormatRule`

Validates a phone number and normalizes it to E.164 format. When normalization succeeds, `correctedValue` is populated — the `DefaultFieldStandardizer` will apply it back to `coreAttributes` before save.

| Param | Required | Description |
|---|---|---|
| `defaultCountryCode` | No | Country dial code to prepend if absent (e.g. `+1`, `+49`) |

```yaml
- rule-id: PHONE_FORMAT
  dimension: VALIDITY
  field-path: phone
  params:
    defaultCountryCode: "+1"
```

#### `DATE_FORMAT`
**Class:** `com.sierra.mdm.dq.rule.validity.DateFormatRule`

Parses a date field against one or more input formats and normalizes to ISO-8601 (`yyyy-MM-dd`). Populates `correctedValue` with the normalized string.

| Param | Required | Description |
|---|---|---|
| `inputFormats` | No | Comma-separated list of accepted formats. Defaults to `MM/dd/yyyy,dd-MM-yyyy,yyyy-MM-dd` |

```yaml
- rule-id: DATE_FORMAT
  dimension: VALIDITY
  field-path: birthDate
  params:
    inputFormats: "MM/dd/yyyy,dd-MM-yyyy,yyyy-MM-dd"
```

---

### Conformity

#### `ALLOWED_VALUES`
**Class:** `com.sierra.mdm.dq.rule.conformity.AllowedValuesRule`

Checks that a field value is a member of a static enumerated set. Case-insensitive by default.

| Param | Required | Description |
|---|---|---|
| `allowedValues` | Yes | Comma-separated list of accepted values |

```yaml
- rule-id: ALLOWED_VALUES
  dimension: CONFORMITY
  field-path: baseUnit
  params:
    allowedValues: "EA,KG,L,M,M2,M3,PC,BOX,PAL"
```

#### `REFERENCE_LOOKUP`
**Class:** `com.sierra.mdm.dq.rule.conformity.ReferenceDataLookupRule`

Validates a field value against a named reference collection in MongoDB. Results are cached in-process via Caffeine to avoid per-record round-trips.

| Param | Required | Description |
|---|---|---|
| `collection` | Yes | MongoDB collection name (e.g. `ref_currency_codes`) |
| `keyField` | Yes | Field in the reference collection to match against |

```yaml
- rule-id: REFERENCE_LOOKUP
  dimension: CONFORMITY
  field-path: currencyCode
  params:
    collection: ref_currency_codes
    keyField: code
```

---

### Consistency

#### `CROSS_FIELD`
**Class:** `com.sierra.mdm.dq.rule.consistency.CrossFieldConsistencyRule`

Evaluates a Spring Expression Language (SpEL) expression over the object's `coreAttributes`. The expression has access to a `coreAttributes` variable (the `Map<String,Object>`). Returns `1.0` if the expression evaluates to `true`, `0.0` otherwise.

All expressions are validated at startup via `@PostConstruct` in `DqEngine`. An invalid expression fails the application context load.

| Param | Required | Description |
|---|---|---|
| `expression` | Yes | SpEL boolean expression |

```yaml
- rule-id: CROSS_FIELD
  dimension: CONSISTENCY
  field-path: _crossField
  params:
    expression: "coreAttributes['country'] != null && coreAttributes['postalCode'] != null"
```

!!! tip "SpEL expression tips"
    Use `coreAttributes['fieldName']` to access core attributes. Use `?:` (Elvis operator) to handle nulls gracefully: `(coreAttributes['startDate'] ?: '9999') <= (coreAttributes['endDate'] ?: '9999')`.

---

### Uniqueness

#### `FIELD_UNIQUE`
**Class:** `com.sierra.mdm.dq.rule.uniqueness.FieldUniquenessRule`

Queries MongoDB for other `ACTIVE` records of the same `objectType` that share the same value for the given field. Returns `1.0` if no duplicates are found, `0.0` otherwise.

!!! warning "Non-blocking by design"
    This rule scores but does **not** throw a validation exception. Hard-blocking on uniqueness during updates would prevent legitimate merge workflows. The score signals the issue to data stewards through the DQ dashboard without preventing the mutation.

```yaml
- rule-id: FIELD_UNIQUE
  dimension: UNIQUENESS
  field-path: email
```

---

## Rule Registry

`DqRuleRegistry` collects all `DqRule` beans at startup:

```java
// com.sierra.mdm.dq.DqRuleRegistry
@Component
@RequiredArgsConstructor
public class DqRuleRegistry {
    private final List<DqRule> rules;   // Spring injects all DqRule beans

    public List<DqRule> getRulesForDimension(DqDimension dimension) { ... }
    public Optional<DqRule> getRuleById(String ruleId) { ... }
}
```

---

## Adding a Custom Rule

1. Create a class in `com.sierra.mdm.dq.rule.<dimension-package>` implementing `DqRule`
2. Annotate it `@Component`
3. Return a stable `ruleId` string (e.g. `"VAT_NUMBER_FORMAT"`)
4. Return the appropriate `DqDimension`
5. Implement `evaluate()` — read the field from `obj.getCoreAttributes()` using `config.getFieldPath()`, return a `DqRuleResult`
6. Add rule config entries under the relevant object types in `application.yml`

```java
@Component
public class VatNumberFormatRule implements DqRule {

    @Override
    public String getRuleId() { return "VAT_NUMBER_FORMAT"; }

    @Override
    public DqDimension getDimension() { return DqDimension.VALIDITY; }

    @Override
    public DqRuleResult evaluate(MasterDataObject obj, DqRuleConfig config) {
        Object value = obj.getCoreAttributes().get(config.getFieldPath());
        if (value == null) return DqRuleResult.skipped(config);
        boolean valid = value.toString().matches("[A-Z]{2}[0-9A-Z]{2,12}");
        return DqRuleResult.of(config, valid, valid ? 1.0 : 0.0,
            valid ? null : "Invalid VAT number format");
    }
}
```

---

## Score Computation

### Per-dimension score

For each dimension, the engine averages the scores of all rules that belong to it:

```
dimension_score(D) = mean( score(r) for r in rules where r.dimension == D )
```

If no rules are configured for a dimension, that dimension is excluded from the composite calculation (its weight is redistributed proportionally).

### Composite score

```
composite = Σ ( dimension_score(D) × weight(D) ) / Σ weight(D)
```

### Score storage

The composite score is written to `metadata.dataQualityScore` as a formatted string (e.g. `"0.87"`) for backward compatibility. Per-dimension scores are stored in `metadata.dimensionScores` as a `Map<String, Double>`.

A full `DqScoreRecord` (with all rule-level results) is asynchronously written to the `dq_score_history` collection.

---

## Configuration Reference

The full DQ config lives under `mdm.data-quality` in `application.yml`:

```yaml
mdm:
  data-quality:
    async: true                          # score persistence is fire-and-forget
    dimension-weights:
      COMPLETENESS: 0.30
      VALIDITY: 0.25
      CONFORMITY: 0.20
      CONSISTENCY: 0.15
      UNIQUENESS: 0.10
    object-type-config:
      <objectType>:
        enrichment:                      # optional derivation rules
          - target-field: fullName
            source-fields: [firstName, lastName]
            template: "{{firstName}} {{lastName}}"
        rules:
          - rule-id: <RULE_ID>
            dimension: <DIMENSION>
            field-path: <fieldName>
            params:
              <key>: <value>
```

See [Standardization & Enrichment](standardization-enrichment.md) for the `enrichment` block schema.
