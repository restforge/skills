# Reference: API Specification

Source: `restforge-handbook/api-spec/` — installed platform version.
Documents all 16 endpoint types, the response envelope, WHERE clause format,
and sort format.

---

## Table of Contents

1. [Endpoint Types](#endpoint-types)
2. [Response Envelope](#response-envelope)
3. [HTTP Status Codes](#http-status-codes)
4. [WHERE Clause Format](#where-clause-format)
5. [Sort Format](#sort-format)
6. [Standard Request Headers](#standard-request-headers)

---

## Endpoint Types

All endpoints use `POST` unless otherwise noted. Base path: `/{resource}/{action}`.

### Standard CRUD

| Endpoint | Method | Path | Success | Notes |
|---|---|---|---|---|
| Create | POST | `/{resource}/create` | 201 | Auto UUID, audit columns populated |
| Read | POST | `/{resource}/read` | 200 | Paginated list with filter/sort |
| First | POST | `/{resource}/first` | 200 | Single record matching WHERE |
| Update | POST | `/{resource}/update` | 200 | Partial update; distributed lock optional |
| Delete | POST | `/{resource}/delete` | 200 | Hard or soft-delete based on SDF |
| Restore | POST | `/{resource}/restore` | 200 | Undo soft-delete; requires soft-delete enabled |
| Adjust | POST | `/{resource}/adjust` | 200 | Atomic numeric increment/decrement with guard |

### Workflow

| Endpoint | Method | Path | Success | Notes |
|---|---|---|---|---|
| Change-Status | POST | `/{resource}/change-status` | 200 | State machine transition; calls onBefore/onAfter hooks |

### Query / Retrieval

| Endpoint | Method | Path | Success | Notes |
|---|---|---|---|---|
| Datatables | POST | `/{resource}/datatables` | 200 | Server-side DataTables.js processing |
| Lookup | GET/POST | `/{resource}/lookup` | 200 | Dropdown/autocomplete; returns `{ id, text }` pairs |
| Aggregate | POST | `/{resource}/aggregate` | 200 | GROUP BY with COUNT/SUM/AVG/MIN/MAX |

### Composite (Master-Detail)

| Endpoint | Method | Path | Success | Notes |
|---|---|---|---|---|
| Create-Composite | POST | `/{resource}/create-composite` | 201 | Master + detail items in one transaction |
| Update-Composite | POST | `/{resource}/update-composite` | 200 | Master + insert/update/delete detail items |
| Read-Composite | POST | `/{resource}/read-composite` | 200 | Master record with nested detail arrays |

### Export / Import

| Endpoint | Method | Path | Success | Notes |
|---|---|---|---|---|
| Export | POST + GET | `/{resource}/export` | 200 | Async: POST triggers, GET downloads .xlsx |
| Import Preview | POST | `/{resource}/import-preview` | 200 | Upload .xlsx, returns validation diff |
| Import Commit | POST | `/{resource}/import-commit` | 200 | Apply previewed import |

### Distributed Lock (manual)

| Endpoint | Method | Path | Notes |
|---|---|---|---|
| Lock Acquire | POST | `/{resource}/lock/acquire` | Lock one or more records; all-or-nothing |
| Lock Release | POST | `/{resource}/lock/release` | Release acquired locks |
| Lock Status | POST | `/{resource}/lock/status` | Check lock state for records |

---

## Response Envelope

All endpoints return a consistent JSON envelope.

```json
{
  "success": true,
  "message": "Record created successfully",
  "data": { ... },
  "timestamp": "2025-01-15T08:30:00.000Z"
}
```

On error:
```json
{
  "success": false,
  "message": "Validation failed",
  "error": "VALIDATION_ERROR",
  "errors": {
    "email": ["Email is required", "Invalid email format"],
    "qty": ["Must be greater than 0"]
  },
  "timestamp": "2025-01-15T08:30:00.000Z"
}
```

| Property | Present when | Notes |
|---|---|---|
| `success` | Always | `true` or `false` |
| `message` | Always | Human-readable summary |
| `data` | Success | Single record, array, or paginated result |
| `timestamp` | Always | ISO 8601 UTC |
| `error` | Failure | Machine-readable error code |
| `errors` | Validation failure | Object keyed by field name, value = array of messages |
| `details` | Server error | Stack trace or extended detail (development mode only) |

**Paginated `data` structure** (for `/read`, `/datatables`):
```json
{
  "data": {
    "rows": [ ... ],
    "total": 150,
    "page": 1,
    "limit": 20
  }
}
```

---

## HTTP Status Codes

| Code | Meaning | Endpoints |
|---|---|---|
| 200 | OK | All read, update, delete, restore, adjust, change-status success |
| 201 | Created | `/create`, `/create-composite` success |
| 400 | Bad Request | Invalid payload, missing required fields, field validation errors |
| 404 | Not Found | Record not found on update, delete, adjust, restore |
| 409 | Conflict | Unique constraint violation; FK constraint on delete; distributed lock race condition |
| 422 | Unprocessable Entity | Invalid workflow status transition; adjust guard failed |
| 500 | Internal Server Error | DB error, unexpected server error |
| 502 | Bad Gateway | `onBefore`/`onAfter` hook returned error or was unreachable |

---

## WHERE Clause Format

The `where` parameter is accepted by `/read`, `/first`, `/datatables`,
`/update`, `/delete`, `/restore`, and `/aggregate`.

**Simple format** — array of AND conditions:
```json
"where": [
  { "key": "status", "value": "active" },
  { "key": "category_id", "value": "abc-123" }
]
```

**Complex format** — explicit logic with operators:
```json
"where": {
  "logic": "OR",
  "conditions": [
    { "key": "status", "operator": "=", "value": "active" },
    { "key": "status", "operator": "=", "value": "pending" }
  ]
}
```

Supported operators:

| Operator | Example value |
|---|---|
| `=` | `"active"` |
| `!=` | `"deleted"` |
| `>` | `100` |
| `<` | `100` |
| `>=` | `0` |
| `<=` | `999` |
| `LIKE` | `"%john%"` |
| `NOT LIKE` | `"%test%"` |
| `BETWEEN` | `[10, 100]` |
| `NOT BETWEEN` | `[10, 100]` |
| `IN` | `["active", "pending"]` |
| `NOT IN` | `["deleted", "archived"]` |
| `IS NULL` | `null` |
| `IS NOT NULL` | `null` |

**Nested logic** — conditions can be nested for complex AND/OR trees:
```json
"where": {
  "logic": "AND",
  "conditions": [
    { "key": "is_active", "operator": "=", "value": true },
    {
      "logic": "OR",
      "conditions": [
        { "key": "role", "operator": "=", "value": "admin" },
        { "key": "role", "operator": "=", "value": "manager" }
      ]
    }
  ]
}
```

**Case-insensitive search** — add `"sensitive": false` to any condition:
```json
{ "key": "email", "operator": "LIKE", "value": "%@gmail.com%", "sensitive": false }
```

---

## Sort Format

The `sort` parameter is accepted by `/read`, `/datatables`, and `/aggregate`.

```json
"sort": [
  { "column": "created_at", "direction": "DESC" },
  { "column": "customer_name", "direction": "ASC" }
]
```

- `direction`: `"ASC"` or `"DESC"`.
- Multiple sort entries are applied in order (primary sort first).
- Column names must match `fieldName` values in the RDF payload.

---

## Standard Request Headers

| Header | Required | Notes |
|---|---|---|
| `Content-Type` | yes | `application/json` for JSON body; `multipart/form-data` for import |
| `Authorization` | yes (if auth enabled) | `Bearer <token>` — JWT from auth module |
| `X-App-Code` | yes (if multi-app) | Identifies which app's scope to apply |
