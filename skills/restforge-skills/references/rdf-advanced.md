# Reference: RDF Advanced Features

Source: `restforge-handbook/catalogs/rdf/` — installed platform version.
This reference covers advanced RDF payload features beyond standard CRUD fields.
For `fieldValidation` constraints, see `references/field-validation.md`.

---

## Table of Contents

1. [Data Source Resolution](#data-source-resolution)
2. [Query File Reference](#query-file-reference)
3. [Field Lookup](#field-lookup)
4. [Default Scope](#default-scope)
5. [Workflow (Change-Status)](#workflow-change-status)
6. [Master-Detail (Composite)](#master-detail-composite)
7. [Aggregate Config](#aggregate-config)
8. [Adjust Config](#adjust-config)
9. [Import Config](#import-config)
10. [Processor](#processor)
11. [Kafka Event Publishing](#kafka-event-publishing)
12. [Components (Lifecycle Hooks)](#components-lifecycle-hooks)

---

## Data Source Resolution

RESTForge resolves data sources per endpoint using a priority chain. Define
only what is needed; the platform falls back automatically.

| Endpoint | Resolution order |
|---|---|
| `/datatables` | `datatablesQuery` → `SELECT * FROM tableName` |
| `/read`, `/first`, `/lookup` | `viewName` → `viewQuery` → `tableName` |
| `/export` | `exportQuery` → `SELECT {fields} FROM tableName` |
| `/read-composite` (detail) | `detailQuery` → detail `tableName` |

**`viewName`** — reference a database VIEW:
```json
"viewName": "v_order_summary"
```

**`viewQuery`** — inline SQL (virtual view, no DB object created):
```json
"viewQuery": "SELECT o.*, c.customer_name FROM orders o JOIN customers c ON o.customer_id = c.customer_id"
```

**`datatablesQuery`** — SQL for the paginated table with `:search`, `:sort`,
`:limit`, `:offset` placeholders:
```json
"datatablesQuery": "SELECT o.*, c.customer_name FROM orders o JOIN customers c ON o.customer_id = c.customer_id WHERE 1=1"
```

Write source is always `tableName` — `viewName`/`viewQuery` are read-only.

---

## Query File Reference

SQL queries can be stored in external `.sql` files using the `file:` prefix.
Path is relative to the payload file location.

```json
"datatablesQuery": "file:sql/orders-datatables.sql",
"exportQuery": "file:sql/orders-export.sql"
```

Convention for folder structure:
```
payload/
├── order.json
└── sql/
    ├── orders-datatables.sql
    └── orders-export.sql
```

External SQL files support the same placeholders as inline queries.
For master-detail, each detail query is in a separate file.

---

## Field Lookup

Configures dropdown/autocomplete data for a field. Used for foreign key fields
that need a human-readable label.

```json
{
  "fieldName": "category_id",
  "type": "string",
  "fieldLookup": {
    "apiPath": "/category",
    "id": "category_id",
    "text": "category_name"
  }
}
```

- `apiPath` — backend resource that provides lookup options via `/lookup` endpoint.
- `id` — field returned as the stored value.
- `text` — field returned as the display label.

Static lookup (no API call):
```json
"fieldLookup": {
  "type": "static",
  "options": [
    { "id": "A", "text": "Option A" },
    { "id": "B", "text": "Option B" }
  ]
}
```

---

## Default Scope

Automatic WHERE clause injected on `/lookup` and `/read`-family endpoints.
Used for tenant isolation, user-scoped data, or active record filtering.

```json
"defaultScope": {
  "actions": ["lookup", "read", "datatables"],
  "conditions": [
    { "key": "is_active", "value": true },
    { "key": "company_id", "value": ":companyId" }
  ]
}
```

- `actions[]` — which endpoints apply the scope.
- Conditions with `:paramName` resolve from the request context (e.g., JWT claims).
- Combines with user-supplied WHERE via AND.
- The `is_active` column is auto-synced by the processor's `.active()` method.

---

## Workflow (Change-Status)

Adds a `/change-status` endpoint with state machine validation.

```json
"workflow": {
  "statusField": "status",
  "transitions": [
    {
      "from": "draft",
      "to": "submitted",
      "action": "submit",
      "onBefore": "http://internal-service/validate",
      "onAfter": "http://notification-service/notify"
    },
    {
      "from": "submitted",
      "to": "approved",
      "action": "approve"
    },
    {
      "from": ["submitted", "approved"],
      "to": "rejected",
      "action": "reject"
    }
  ]
}
```

| Property | Notes |
|---|---|
| `statusField` | Field name that holds the current status |
| `transitions[].from` | Current status (string or array of strings) |
| `transitions[].to` | Target status after transition |
| `transitions[].action` | Action identifier in the request body |
| `transitions[].onBefore` | HTTP call before transition; 4xx/5xx blocks the transition (HTTP 422/502) |
| `transitions[].onAfter` | HTTP call after transition; failure does not roll back |

---

## Master-Detail (Composite)

Adds `/create-composite`, `/update-composite`, and `/read-composite` endpoints.

```json
"details": [
  {
    "tableName": "order_item",
    "foreignKey": "order_id",
    "primaryKey": "item_id",
    "fields": [
      { "fieldName": "product_id", "type": "string" },
      { "fieldName": "qty", "type": "integer" },
      { "fieldName": "price", "type": "decimal" }
    ]
  }
]
```

- Multiple entries in `details[]` generate multiple detail tabs.
- `foreignKey` links detail rows to the master record.
- Detail `fields[]` follow the same validation rules as master fields.
- `/update-composite` supports three detail operations in one call:
  `insert` (new rows), `update` (changed rows), `delete` (removed rows).
- `/read-composite` returns the master record with all detail arrays nested.

---

## Aggregate Config

Adds an `/aggregate` endpoint for COUNT, SUM, AVG, MIN, MAX operations.

```json
"aggregateConfig": {
  "joins": [
    {
      "type": "LEFT",
      "table": "category",
      "on": "product.category_id = category.category_id"
    }
  ],
  "groupBy": ["category_name"],
  "operations": [
    { "function": "COUNT", "field": "product_id", "alias": "total_products" },
    { "function": "SUM", "field": "stock_qty", "alias": "total_stock" },
    { "function": "AVG", "field": "price", "alias": "avg_price" }
  ]
}
```

| Operation | Description |
|---|---|
| `COUNT` | Count rows or non-null values |
| `SUM` | Sum numeric field |
| `AVG` | Average numeric field |
| `MIN` | Minimum value |
| `MAX` | Maximum value |

The client sends `groupBy[]` and `having[]` in the request to filter results.

---

## Adjust Config

Adds an `/adjust` endpoint for atomic numeric field increments/decrements.
Prevents race conditions on stock, balance, and counter fields.

```json
"adjustConfig": {
  "fields": ["stock_qty", "reserved_qty"],
  "guards": [
    {
      "field": "stock_qty",
      "operator": "gte",
      "value": 0,
      "message": "Stock cannot be negative"
    }
  ]
}
```

- `fields[]` — fields that can be adjusted; must be numeric type.
- `guards[]` — pre-condition checks; request is rejected (HTTP 422) if any
  guard fails after applying the adjustment.
- The client sends `{ "field": "stock_qty", "amount": -5 }` in the request.
- Adjustment is executed as an atomic SQL UPDATE with WHERE guard.

---

## Import Config

Adds `/import-preview` and `/import-commit` endpoints for Excel (.xlsx) imports.

```json
"importConfig": {
  "sheet": 0,
  "startRow": 2,
  "strategy": "upsert",
  "upsertKey": ["sku"],
  "columns": [
    { "header": "SKU", "fieldName": "sku" },
    { "header": "Product Name", "fieldName": "product_name" },
    {
      "header": "Category",
      "fieldName": "category_id",
      "lookup": { "apiPath": "/category", "matchField": "category_name", "returnField": "category_id" }
    }
  ]
}
```

| Property | Notes |
|---|---|
| `sheet` | Sheet index (0-based) or sheet name |
| `startRow` | First data row (1-based); default: `2` (row 1 = header) |
| `strategy` | `"insert"` (fail on duplicate) or `"upsert"` (update on match) |
| `upsertKey[]` | Fields used to identify existing records for upsert |
| `columns[].header` | Excel column header text |
| `columns[].fieldName` | RDF field to map to |
| `columns[].lookup` | Resolve a display value to an ID before insert |

Import is a two-step process: `/import-preview` validates and returns a diff;
`/import-commit` applies changes. The client uploads the Excel file to `/import-preview`
with a `POST multipart/form-data` request.

---

## Processor

Alternative RDF structure for custom non-CRUD endpoints. A processor payload
does NOT have `tableName`, `fieldName`, or `action`. Each entry in `processor[]`
defines one endpoint. Generated with `npx restforge processor create`.

```json
{
  "description": "Sales Order custom endpoints",
  "processor": [
    {
      "name": "submit-order",
      "method": "POST",
      "description": "Submit draft order menjadi pending approval",
      "sql": {
        "query": "UPDATE sales.sales_order SET status = 'pending_approval' WHERE so_id = $1 AND status = 'draft'",
        "params": ["so_id"]
      },
      "request": {
        "body": {
          "so_id": { "type": "uuid", "required": true },
          "notes": { "type": "string", "required": false, "maxLength": 200 }
        },
        "headers": {
          "X-App-Code": { "type": "string", "required": true, "mapTo": "app_code" }
        }
      },
      "response": {
        "message": {
          "success": "Sales order berhasil di-submit untuk approval.",
          "empty":   "Sales order tidak ditemukan atau bukan berstatus draft.",
          "error":   "Gagal submit sales order."
        }
      }
    }
  ]
}
```

| Property | Required | Notes |
|---|---|---|
| `processor[].name` | Yes | Endpoint name — becomes file name and URL segment |
| `processor[].method` | Yes | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `processor[].sql.query` | Conditional | Inline SQL with `$1, $2, ...` placeholders. If `sql` block present, one of `query` or `file` is required |
| `processor[].sql.file` | Conditional | Path to external `.sql` file, relative to payload folder |
| `processor[].sql.params` | No | Field names bound to placeholders; resolved from body/params/query/header `mapTo`; falls back to `default` if input empty |
| `processor[].request.body` | No | Request body field schema |
| `processor[].request.params` | No | Route params; each key adds `/:key` to the path |
| `processor[].request.headers` | No | Header schema; use `mapTo` to rename into `input` |
| `processor[].request.validate` | No | Default `true`. Set `false` to opt-out router-level validation |
| `processor[].response.message` | No | `success`, `empty` (SQL mode only), `error` messages |
| `processor[].cache.enabled` | No | Default `false`. Response cache for GET processors |
| `processor[].cache.ttl` | No | Cache TTL in seconds; default `300` |

**Field schema properties** (apply to `request.body`, `request.params`, `request.headers`):

| Property | Notes |
|---|---|
| `type` | `string`, `number`, `integer`, `boolean`, `uuid`, `array`, `object`, `date`, `datetime` |
| `required` | Router rejects with HTTP 400 if absent |
| `format` | Regex whitelist: `email`, `url`, `phone-id`, `uuid` |
| `enum` | Whitelist of allowed values |
| `minLength` / `maxLength` | Length check for string fields |
| `sensitive` | `true` masks value as `***MASKED***` in router debug log |
| `default` | Fallback value for `sql.params` binding when input is empty |
| `mapTo` | (`headers` only) field name to use in `input` object |

**Generator behavior:**
- Router (`{endpoint}.js`) — always overwritten on re-run.
- Processor file (`processor/{endpoint}/{name}.js`) — skipped if already exists (safe to re-run).
  Use `--force` to overwrite.

**Without sql block** — payload with only `name`, `method`, and `request` is valid.
Router registers the route with validation; processor file is generated as a manual
implementation scaffold. Business logic is written in the processor file.

---

## Kafka Event Publishing

Publishes events to a Kafka topic after CRUD operations. Requires
`KAFKA_ENABLED=true` in backend config.

```json
"kafka": {
  "events": [
    { "action": "create", "topic": "order.created.events" },
    { "action": "update", "topic": "order.updated.events" },
    { "action": "delete", "topic": "order.deleted.events" }
  ]
}
```

- `action` — CRUD action that triggers the event: `create`, `update`, `delete`,
  `change-status`, `create-composite`, `update-composite`.
- `topic` — Kafka topic name. Supports `{module}` and `{endpoint}` placeholders:
  `"{module}.{endpoint}.events"`.
- Event payload contains the full record after the operation.
- Publishing is async and does not block the API response.

---

## Components (Lifecycle Hooks)

`components` configures CRUD lifecycle hooks that execute local JavaScript handler
files. Added to a standard CRUD payload (one that has `tableName`).

```json
{
  "components": [
    {
      "properties": {
        "filename": "components/supplier-hooks.js",
        "methods": [
          {
            "name": "validateSupplierCode",
            "events": "onBeforeInsert",
            "params": [
              { "value": "{requestData}" },
              { "value": "{user_id}" }
            ]
          },
          {
            "name": "notifySlack",
            "events": "onAfterInsert"
          }
        ]
      }
    }
  ]
}
```

| Property | Required | Notes |
|---|---|---|
| `components[].properties.filename` | Yes | Path to handler file, relative to project root |
| `components[].properties.methods` | Yes | List of method bindings |
| `methods[].name` | Yes | Function name exported from the handler file |
| `methods[].events` | Yes | Event hook (see table below) |
| `methods[].params` | No | Template variables forwarded to the handler |

**Supported event hooks:**

| Event | Trigger |
|---|---|
| `onBeforeInsert`, `onAfterInsert` | `/create` endpoint |
| `onBeforeUpdate`, `onAfterUpdate` | `/update` endpoint |
| `onBeforeDelete`, `onAfterDelete` | `/delete` endpoint |
| `onBeforeCompositeInsert`, `onAfterCompositeInsert` | `/create-composite` endpoint |
| `onBeforeCompositeUpdate`, `onAfterCompositeUpdate` | `/update-composite` endpoint |

**Template variables for `params[].value`:**

| Variable | Value |
|---|---|
| `{tableName}` | Resource table name |
| `{requestData}` | Full request body |
| `{oldData}` | Data before operation (`update`, `delete`) |
| `{newData}` | Data after operation (`create`, `update`) |
| `{operation}` | Operation name: `insert` / `update` / `delete` |
| `{user_id}` | User ID from request context |
| `{timestamp}` | Execution timestamp |
| `{record_id}` | Primary key of the affected record |

**Handler file signature** (`src/components/handlers/`):

```javascript
async function handlerName(/* resolved params... */, services) {
  const { db, logger, redis, kafka, cache } = services;
  // business logic
  return { success: true, message: '...' };
}
module.exports = { handlerName };
```

- `services` is injected automatically as the last argument; no need to declare it
  in `params[]`.
- All events are **blocking** — `return { success: false }` or throwing an exception
  rolls back the entire transaction.
- If `components` is absent from the payload, CRUD operates normally without hooks.
