# Reference: dbschema-kit Catalog

Source: `codegen_get_dbschema_catalog` — installed platform version.
Use as grounding before defining SDF.
Schema version: 1.1.

---

## Table of Contents

1. [Field Types](#field-types)
2. [Shorthand Syntax](#shorthand-syntax)
3. [Constraints](#constraints)
4. [Relation Types](#relation-types)
5. [Referential Actions](#referential-actions)
6. [Check Operations](#check-operations)
7. [Audit Columns](#audit-columns)
8. [Soft-Delete Contract](#soft-delete-contract)
9. [Naming Rules](#naming-rules)
10. [Dialect Support](#dialect-support)

---

## Field Types

10 types available. `string` and `decimal` require a modifier.

| Type | Modifier | Shorthand example | Notes |
|---|---|---|---|
| `string` | required: `:<length>` | `string:255` | VARCHAR, explicit length required |
| `text` | none | `text` | TEXT/CLOB, no length limit |
| `integer` | none | `integer` | INT 32-bit |
| `bigint` | none | `bigint` | BIGINT 64-bit |
| `decimal` | required: `:<precision>,<scale>` | `decimal:15,2` | Fixed-point |
| `boolean` | none | `boolean` | Native BOOLEAN (PG), VARCHAR on others |
| `date` | none | `date` | Date only |
| `timestamp` | none | `timestamp` | Date + time |
| `uuid` | none | `uuid` | Native UUID (PG), VARCHAR(36) on others |
| `json` | none | `json` | JSONB (PG), JSON (MySQL), CLOB (Oracle) |

---

## Shorthand Syntax

Format: `<type>[:<modifier>] [<constraint>[:<value>]]...`

```
string:36 pk
string:255 notnull
decimal:15,2 notnull default:0
boolean default:true
string:100 default:'pending'
timestamp default:now()
string:36 fk:category.id
string:64 index
```

Rules:
- Type must come first and must be from the field types list above.
- Modifier is required for `string` (length) and `decimal` (precision,scale).
- Standalone constraints have no value: `pk`, `notnull`, `unique`, `index`.
- Value constraints require `constraint:value` format: `default`, `fk`.
- String default: single-quoted `default:'value'`; numeric/boolean: raw `default:0`;
  SQL constant: bare identifier `default:current_date`; native function: `default:now()`.
- FK uses dot notation: `fk:<table>.<column>`. Bracket syntax is rejected by the parser.
- `autoUpdate` is deprecated — no functional effect, parsed for backward compat only.

---

## Constraints

7 constraints available (excludes deprecated `autoUpdate`).

| Constraint | Kind | Example | Notes |
|---|---|---|---|
| `pk` | standalone | `string:36 pk` | Primary key |
| `notnull` | standalone | `string:255 notnull` | NOT NULL |
| `unique` | standalone | `string:32 unique` | Single-column UNIQUE |
| `index` | standalone | `string:64 index` | Single-column non-unique index |
| `default` | value | `boolean default:true` | Default value |
| `fk` | value | `string:36 fk:category.id` | FK + auto belongsTo relation |

---

## Relation Types

3 relation types. All use `type`, `localKey`, `references` (required),
and `target`, `onDelete`, `onUpdate` (optional).

| Type | Direction | DDL |
|---|---|---|
| `belongsTo` | Many-to-one, this table holds the FK | FK constraint generated |
| `hasOne` | One-to-one inverse of belongsTo | No additional DDL |
| `hasMany` | One-to-many inverse of belongsTo | No additional DDL |

Example:
```js
relations: {
  category: {
    type: "belongsTo",
    localKey: "category_id",
    references: "category_id",
    onDelete: "restrict"
  }
}
```

---

## Referential Actions

4 actions for `onDelete` and `onUpdate`:

| Action | Behavior |
|---|---|
| `cascade` | Delete/update child rows |
| `restrict` | Reject delete/update if child exists |
| `setNull` | Set child FK to NULL (field must be nullable) |
| `noAction` | Defer check (semantics similar to restrict in most dialects) |

---

## Check Operations

7 operators for the `checks` array. Format: `{ name?, field, <operator>: <value> }`.

| Operator | Value type | Example |
|---|---|---|
| `in` | array | `{ field: "status", in: ["active", "inactive"] }` |
| `eq` | scalar | `{ field: "type", eq: "user" }` |
| `neq` | scalar | `{ field: "type", neq: "system" }` |
| `gt` | numeric | `{ field: "qty", gt: 0 }` |
| `gte` | numeric | `{ field: "qty", gte: 0 }` |
| `lt` | numeric | `{ field: "discount", lt: 100 }` |
| `lte` | numeric | `{ field: "discount", lte: 100 }` |

---

## Audit Columns

4 standard columns for tables managed by RESTForge. Emitted automatically by
`codegen_dbschema_init`. Lookup/system tables may remove them manually.

| Column | SDF shorthand | Notes |
|---|---|---|
| `created_at` | `timestamp default:now()` | Set via DEFAULT now() on INSERT |
| `created_by` | `string:100` | Set by application layer on INSERT, nullable |
| `updated_at` | `timestamp` | Auto-updated via RDF runtime (not an SDF marker) |
| `updated_by` | `string:100` | Set by application layer on UPDATE, nullable |

All 4 columns are nullable. Drift between SDF (audit columns absent) and RDF
(assumes audit columns exist) causes runtime errors.

---

## Soft-Delete Contract

Declare with `softDelete: { enabled: true }` in `defineModel`. The three
contract columns must be present in `fields` with the correct types.

| Column | Required shorthand | Nullable |
|---|---|---|
| `is_deleted` | `boolean notnull default:false` | no |
| `deleted_at` | `timestamp` | yes |
| `deleted_by` | `string:70` | yes |

### Biconditional rules

- `enabled: true` + all three columns present with correct types → valid.
- `enabled: true` + missing columns → ERROR (missing columns listed).
- Contract columns present but `enabled` not true → ERROR.
- Wrong column type → ERROR.
- `is_deleted`, `deleted_at`, `deleted_by` are reserved names — cannot be used
  as regular columns in a RESTForge table.

### Reusable (UNIQUE column values freed after soft-delete)

Fields whose unique values should be freed after soft-delete must be declared
in `softDelete.reusable: [{ field, length }]`. Requirements:
- Field must exist in `fields`.
- Field type must be `string` or `text`.
- Field must have a single-column UNIQUE constraint.
- `length` = maximum input length. Physical length = `length + 38`
  (38 = "##" + UUID v7).

### Generated DDL

- CHECK constraint: `chk_<table>_soft_delete_consistency` — enforces consistency
  across the three contract columns.
- Non-unique indexes become PostgreSQL partial indexes:
  `WHERE is_deleted = FALSE`.
- UNIQUE constraints are NOT made partial; column values are freed via
  suffix mutation (not a partial unique index).

### Unique eligibility gate

When `enabled: true`, all UNIQUE constraints on the table are checked:
- Composite UNIQUE is rejected — suffix mutation cannot free composite columns.
- Single-column UNIQUE on a non-string type is rejected — suffix mutation only
  works for `string`/`text`.

### Dialect support

Phase 1: PostgreSQL only. Other dialects produce an explicit error.

---

## Naming Rules

- Table names: `snake_case`, lowercase, digits, underscores.
- Field/column names: `snake_case`.
- Constraint names: auto-generated with prefixes `pk_`, `fk_`, `idx_`, `uq_`,
  `ck_`. When exceeding the dialect's maximum length, falls back to an 8-character
  MD5 suffix.

---

## Dialect Support

| Dialect | Driver | Boolean storage |
|---|---|---|
| `postgres` | pg | Native BOOLEAN |
| `mysql` | mysql2 | VARCHAR ("true"/"false") |
| `oracle` | oracledb | VARCHAR2 + CHECK constraint |
| `sqlite` | better-sqlite3 | TEXT |
