# Reference: Field Validation Catalog

Source: `codegen_get_field_validation_catalog` — installed platform version.
Use as grounding before defining `fieldValidation` in a payload.
Schema version: 1.0.

Summary: 12 types, 32 constraints, 4 format presets.

---

## Table of Contents

1. [Types and Applicable Constraints](#types-and-applicable-constraints)
2. [Constraints (full)](#constraints-full)
3. [Format Presets](#format-presets)
4. [Audit Columns in Payload](#audit-columns-in-payload)
5. [Message Override Pattern](#message-override-pattern)

---

## Types and Applicable Constraints

Use the `applicableConstraints` column to validate constraint scope per type.
Constraints not listed for a given type will be rejected.

| Type | Database Types | Applicable Constraints |
|---|---|---|
| `string` | VARCHAR, TEXT, CHAR | required, unique, default, primaryKey, autoGenerate, nullable, minLength, maxLength, pattern, patternMessage, format, enum, trim, lowercase, uppercase |
| `integer` | INTEGER, INT, BIGINT | required, unique, default, primaryKey, nullable, min, max, precision, scale, positive, negative, integer |
| `decimal` | DECIMAL, NUMERIC | required, unique, default, primaryKey, nullable, min, max, precision, scale, positive, negative, integer |
| `number` | NUMERIC | required, unique, default, primaryKey, nullable, min, max, precision, scale, positive, negative, integer |
| `boolean` | BOOLEAN | required, unique, default, primaryKey, nullable, strict |
| `date` | DATE | required, unique, default, primaryKey, autoGenerate, nullable, format, min, max, before, after |
| `datetime` | TIMESTAMP | required, unique, default, primaryKey, autoGenerate, nullable, format, min, max, before, after |
| `timestamp` | TIMESTAMPTZ | required, unique, default, primaryKey, autoGenerate, nullable, format, min, max, before, after |
| `time` | TIME | required, unique, default, primaryKey, nullable |
| `uuid` | UUID | required, unique, default, primaryKey, autoGenerate, nullable |
| `json` | JSON, JSONB | required, unique, default, primaryKey, nullable, schema |
| `array` | ARRAY | required, unique, default, primaryKey, nullable, minItems, maxItems, uniqueItems |

---

## Constraints (full)

### General (applies to all types)

| Constraint | Value type | Notes |
|---|---|---|
| `required` | boolean | Field must be present and non-empty |
| `unique` | boolean | Unique value across rows (enforced by DB, not app validation) |
| `default` | any | Default value when field is absent |
| `primaryKey` | boolean | Mark as primary key |
| `autoGenerate` | boolean | Auto-generate value at runtime (uuid, string, timestamp, datetime, date) |
| `nullable` | boolean | Allow null values |

### String scope

| Constraint | Value type | Message override key | Example |
|---|---|---|---|
| `minLength` | integer | `minLengthMessage` | `"minLength": 3` |
| `maxLength` | integer | `maxLengthMessage` | `"maxLength": 100` |
| `pattern` | string | `patternMessage` | `"pattern": "^[A-Z]{3}\\d{4}$"` |
| `patternMessage` | string | — | `"patternMessage": "Invalid format"` |
| `format` | string | `formatMessage` | `"format": "email"` (see format presets) |
| `enum` | array | `enumMessage` | `"enum": ["active", "inactive"]` |
| `trim` | boolean | — | `"trim": true` |
| `lowercase` | boolean | — | `"lowercase": true` |
| `uppercase` | boolean | — | `"uppercase": true` |

### Number scope (integer, decimal, number)

| Constraint | Value type | Message override key | Example |
|---|---|---|---|
| `min` | number | `minMessage` | `"min": 0` |
| `max` | number | `maxMessage` | `"max": 9999999.99` |
| `precision` | integer | `precisionMessage` | `"precision": 10` |
| `scale` | integer | — | `"scale": 2` |
| `positive` | boolean | `positiveMessage` | `"positive": true` |
| `negative` | boolean | `negativeMessage` | `"negative": true` |
| `integer` | boolean | `integerMessage` | `"integer": true` |

### Date scope (date, datetime, timestamp)

| Constraint | Value type | Message override key | Example |
|---|---|---|---|
| `format` | string | — | `"format": "DD/MM/YYYY"` |
| `min` | string | `minMessage` | `"min": "01/01/2020"` |
| `max` | string | `maxMessage` | `"max": "31/12/2030"` |
| `before` | string (field name) | `beforeMessage` | `"before": "end_date"` |
| `after` | string (field name) | `afterMessage` | `"after": "start_date"` |

### Boolean scope

| Constraint | Value type | Notes |
|---|---|---|
| `strict` | boolean | Reject coercion — only accept native boolean values, not the strings "true"/"false" |

### Array scope

| Constraint | Value type | Message override key | Example |
|---|---|---|---|
| `minItems` | integer | `minItemsMessage` | `"minItems": 1` |
| `maxItems` | integer | `maxItemsMessage` | `"maxItems": 100` |
| `uniqueItems` | boolean | `uniqueItemsMessage` | `"uniqueItems": true` |

### JSON scope

| Constraint | Value type | Example |
|---|---|---|
| `schema` | object | `"schema": { "type": "object", "properties": { ... } }` |

---

## Format Presets

Applies to the `format` constraint on type `string`.

| Preset | Notes |
|---|---|
| `email` | Validates email address format |
| `phone` | Validates phone number format |
| `url` | Validates URL format |
| `uuid` | Validates UUID format (v7 generated by app layer; v4 legacy remains valid) |

---

## Audit Columns in Payload

The `auditColumns` key in a payload controls which audit columns are managed by the runtime.

| Value | Behavior |
|---|---|
| absent (no key) | Use 4 default audit columns: created_at, created_by, updated_at, updated_by |
| `false` | Disable audit columns |
| `null` | Disable audit columns |
| object | Override column names; required keys: `createdAt`, `createdBy`, `updatedAt`, `updatedBy` |

Rejected values: `true`, string, array, number. Error message:
`"Invalid auditColumns value for <tableName>: must be false, null, or object"`.

Auto-update of `updated_at` is implemented entirely in the RDF runtime (BaseModel
auditColumns helper) based on naming convention — not by an SDF marker.

---

## Message Override Pattern

Any constraint that has a `messageOverrideKey` can be overridden by adding a
sibling key named `{constraintName}Message`.

Example:
```json
{
  "fieldName": "supplier_code",
  "type": "string",
  "fieldValidation": [
    { "required": true, "requiredMessage": "Supplier code is required" },
    { "minLength": 3, "minLengthMessage": "Minimum 3 characters" }
  ]
}
```
