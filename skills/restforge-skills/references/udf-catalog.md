# Reference: UDF Catalog (UI Definition File)

> **Offline mirror.** This file mirrors `designer_get_udf_catalog` from the
> installed RESTForge designer. The live tool is authoritative — when this file
> and the tool disagree, trust the tool, then update this file. Field types and
> features depend on the active plugin version; always re-ground with the tool
> (and `designer_list_plugins`) before defining a UDF payload.

Source: `designer_get_udf_catalog` — installed designer version.
Use as grounding before defining or editing any UDF payload.

---

## Table of Contents

1. [Payload Envelope](#payload-envelope)
2. [App Config](#app-config)
3. [Page Anatomy](#page-anatomy)
4. [Field Types](#field-types)
5. [Field Attributes](#field-attributes)
6. [Field Rows (Layout)](#field-rows-layout)
7. [Field States (Conditional)](#field-states-conditional)
8. [Features](#features)
9. [Data Source](#data-source)
10. [Navigation](#navigation)
11. [Homepage](#homepage)
12. [Dashboard Page](#dashboard-page)
13. [Master-Detail](#master-detail)
14. [Workflow Actions](#workflow-actions)
15. [ID Generation](#id-generation)
16. [Live Sync](#live-sync)
17. [Naming Conventions](#naming-conventions)
18. [Plugins](#plugins)

---

## Payload Envelope

Root-level UDF structure. All properties at the top level.

```json
{
  "appConfig": { ... },
  "pages": [ ... ],
  "navigation": [ ... ],
  "homepage": "page-id"
}
```

**Multi-file support:**
- `extends`: inherit from a base file — `"extends": "./base.json"`
- `include`: inline another file's pages array — `"include": "./orders-pages.json"`

---

## App Config

| Property | Type | Required | Notes |
|---|---|---|---|
| `appName` | string | yes | Display name shown in page title and header |
| `appCode` | string | yes | Unique app identifier, kebab-case, used in file naming |
| `plugin` | string | yes | Output plugin: `vanilla-js-basic`, `vanilla-js-auth`, or custom |
| `apiBaseUrl` | string | yes | Base URL of the RESTForge backend API |
| `port` | integer | no | Local dev server port; default: `3000` |
| `numberFormat` | string | no | Locale for number display; default: `"id-ID"` |

Plugin-specific properties (for `vanilla-js-auth`):
- `authAppCode` — app code used for the auth API path
- `idleTimeout` — session idle timeout in seconds

---

## Page Anatomy

Each entry in `pages[]` represents one page in the application.

| Property | Type | Required | Notes |
|---|---|---|---|
| `pageId` | string | yes | Unique identifier; CRUD: `^[a-zA-Z0-9_-]+$`; dashboard: `^dash-[a-z0-9-]+$` |
| `pageTitle` | string | yes | Display title shown in header and sidebar |
| `pageSubtitle` | string | no | Subtitle shown below title |
| `pageIcon` | string | no | Icon class (e.g., Tabler Icons: `"ti ti-home"`) |
| `pageGroup` | string | no | Sidebar group name (Title Case); auto-derives navigation |
| `pageType` | string | no | `"crud"` (default) or `"dashboard"` |
| `apiPath` | string | yes (CRUD) | Backend endpoint path; e.g., `"/customer"` |
| `primaryKey` | string | yes (CRUD) | Primary key field name; e.g., `"customer_id"` |
| `displayField` | string | yes (CRUD) | Field used as record label in lookups |
| `fields` | array | yes (CRUD) | Field definitions for the page |
| `fieldRows` | array | no | Layout grouping for multi-column forms |
| `fieldStates` | array | no | Conditional visibility/readonly rules |
| `details` | array | no | Detail pages for master-detail pattern |
| `workflow` | object | no | Workflow state machine config |
| `workflowActions` | array | no | Action buttons for workflow transitions |
| `features` | object | no | Page-level feature flags |

---

## Field Types

8 field types. Each type maps to a specific HTML element and optional library.

| Type | HTML Element | Library | Key attributes |
|---|---|---|---|
| `text` | `<input type="text">` | — | `maxlength`, `placeholder` |
| `textarea` | `<textarea>` | — | `rows`, `maxlength` |
| `number` | `<input type="number">` | — | `min`, `max`, `step` |
| `checkbox` | toggle switch | — | `defaultValue` (true/false), `checkboxText` |
| `select` | `<select>` | Select2 | `dataSource`, `tableField` |
| `date` | date picker | Flatpickr | `minDate`, `maxDate`, `dateFormat` |
| `timestamp` | datetime picker | Flatpickr | `minDate`, `maxDate`, `dateFormat` |
| `time` | time picker | Flatpickr | `minTime`, `maxTime` |

**Notes:**
- `select` requires a `dataSource` to populate options.
- `date`/`timestamp`/`time` require Flatpickr assets bundled by the plugin.
- `checkbox` renders as a toggle switch, not a standard checkbox input.

---

## Field Attributes

Common attributes for all field types.

| Attribute | Type | Notes |
|---|---|---|
| `name` | string | Field name matching the RDF payload field; snake_case |
| `label` | string | Display label shown on form and table header |
| `type` | string | One of the 8 field types above |
| `required` | boolean | Frontend validation — marks field as mandatory |
| `readonly` | boolean | Renders as read-only input |
| `inTable` | boolean | Show column in the data table; default: `true` |
| `tableOrder` | integer | Column order in table; lower = leftmost |
| `tableField` | string | Column value to display in table (for `select` — show label, not id) |
| `width` | integer | Column width in pixels |
| `placeholder` | string | Input placeholder text |
| `defaultValue` | any | Default value on new record form |
| `editorMode` | string | Form only (`"form"`) or table only (`"table"`) |
| `maxlength` | integer | Max character length for `text`/`textarea` |
| `rows` | integer | Row height for `textarea` |

---

## Field Rows (Layout)

`fieldRows[]` groups fields into a CSS Grid row for multi-column forms.
Without `fieldRows`, fields render in a single column.

```json
"fieldRows": [
  { "fields": ["first_name", "last_name"] },
  { "fields": ["email", "phone", "status"] }
]
```

- Each entry in `fields[]` refers to a field `name` defined in `fields[]`.
- Fields within a row share the available width equally by default.
- Use `colSpan` on a field attribute to span multiple columns:
  `"colSpan": 2` makes a field take 2 of the row's grid columns.
- Interacts with `features.fieldLayout`: `"horizontal"` places the label
  to the left of the input; `"vertical"` (default) stacks them.

---

## Field States (Conditional)

`fieldStates[]` defines conditional visibility and readonly rules based on
field values. Applied at runtime when field values change.

```json
"fieldStates": [
  {
    "when": { "field": "status", "value": "approved" },
    "fields": ["approved_by", "approved_at"],
    "state": "readonly"
  },
  {
    "when": { "field": "order_type", "value": "internal" },
    "fields": ["customer_id"],
    "state": "hidden"
  }
]
```

| Property | Notes |
|---|---|
| `when.field` | The triggering field name |
| `when.value` | The value that triggers the state change |
| `fields[]` | Fields to affect when the condition is met |
| `state` | `"readonly"` or `"hidden"` |

Global scope: `fieldStates` at page level applies to all form instances.

---

## Features

`features` is an object at page level controlling toolbar and layout behavior.

| Feature | Type | Notes |
|---|---|---|
| `enableSearch` | boolean | Show full-text search input in toolbar |
| `enableStatusFilter` | object | Dropdown filter for boolean/enum field; requires `field` and `options[]` |
| `enableDataFilter` | array | Dynamic dropdown filters; each entry requires `field` and `dataSource` |
| `enableLiveSync` | boolean | Enable WebSocket live data refresh; requires `liveSync` at root level |
| `fieldLayout` | string | Form field label placement: `"vertical"` (default) or `"horizontal"` |

`enableStatusFilter` example:
```json
"enableStatusFilter": {
  "field": "is_active",
  "options": [
    { "label": "Active", "value": "true" },
    { "label": "Inactive", "value": "false" }
  ]
}
```

`enableDataFilter` example:
```json
"enableDataFilter": [
  { "field": "category_id", "dataSource": { "type": "api", "apiPath": "/category" } }
]
```

---

## Data Source

Used by `select` fields and `enableDataFilter` to populate options.

**Type: static**
```json
"dataSource": {
  "type": "static",
  "options": [
    { "id": "active", "text": "Active" },
    { "id": "inactive", "text": "Inactive" }
  ]
}
```

**Type: api**
```json
"dataSource": {
  "type": "api",
  "apiPath": "/category",
  "id": "category_id",
  "text": "category_name"
}
```

- `apiPath` calls the backend `/lookup` endpoint of that resource.
- `id` maps to the option value; `text` maps to the display label.
- The `tableField` attribute on the field should reference `text` to show
  the label (not the id) in the data table column.

---

## Navigation

`navigation[]` defines the sidebar menu. Three item types: `page`, `group`,
`separator`.

```json
"navigation": [
  { "type": "page", "pageId": "customer", "icon": "ti ti-users" },
  { "type": "separator" },
  {
    "type": "group",
    "label": "Master Data",
    "icon": "ti ti-database",
    "items": [
      { "type": "page", "pageId": "category" },
      { "type": "page", "pageId": "supplier" }
    ]
  }
]
```

- Maximum nesting depth: 3 levels.
- When `pageGroup` is set on a page, the sidebar auto-derives navigation —
  manual `navigation[]` is optional.
- Manual `navigation[]` overrides auto-derivation completely.
- `icon` uses Tabler Icons class format: `"ti ti-<name>"`.

---

## Homepage

`homepage` sets the default landing page when the app loads.

```json
"homepage": "dashboard-main"
```

- Value must match a `pageId` in `pages[]`.
- Supports both CRUD and dashboard pages.
- When absent, the first page in `navigation[]` is loaded.

---

## Dashboard Page

A page with `pageType: "dashboard"` renders a grid-based analytics view.

```json
{
  "pageId": "dash-overview",
  "pageType": "dashboard",
  "pageTitle": "Overview",
  "dataSources": [
    { "id": "ds-orders", "apiPath": "/order", "action": "aggregate" }
  ],
  "rows": [
    {
      "columns": [
        { "colSpan": { "xl": 4, "lg": 6, "md": 12 }, "widget": { ... } },
        { "colSpan": { "xl": 8, "lg": 6, "md": 12 }, "widget": { ... } }
      ]
    }
  ]
}
```

**7 widget types:**

| Widget type | Display | Key properties |
|---|---|---|
| `column` | Vertical bar chart | `dataSource`, `labelField`, `valueField`, `color` |
| `bar` | Horizontal bar chart | `dataSource`, `labelField`, `valueField`, `color` |
| `pie` | Pie / donut chart | `dataSource`, `labelField`, `valueField` |
| `area` | Area chart | `dataSource`, `labelField`, `valueField`, `color` |
| `mini-bar` | Compact bar (KPI card) | `dataSource`, `valueField`, `label`, `icon`, `color` |
| `progress` | Progress bar list | `dataSource`, `labelField`, `valueField`, `maxValue` |
| `list` | Simple data list | `dataSource`, `columns[]` |

`dataSources[]` at page level defines reusable data pools; widgets reference
them by `id`. Each data source calls the backend `/aggregate` endpoint.

---

## Master-Detail

A page with `details[]` renders a master-detail composite form.

```json
{
  "pageId": "sales-order",
  "apiPath": "/order",
  "primaryKey": "order_id",
  "displayField": "order_no",
  "fields": [ ... ],
  "details": [
    {
      "pageId": "order-item",
      "apiPath": "/order-item",
      "foreignKey": "order_id",
      "primaryKey": "item_id",
      "displayField": "product_name",
      "fields": [ ... ]
    }
  ]
}
```

- Each entry in `details[]` is a sub-page bound to the master via `foreignKey`.
- Multiple detail tabs are supported — one entry per detail entity.
- Generates composite form: master header + detail items grid.
- Backend must have composite endpoints (`/create-composite`, etc.) generated
  from the RDF master-detail payload.

---

## Workflow Actions

`workflowActions[]` adds status-transition buttons to the form toolbar.

```json
"workflow": {
  "statusField": "status"
},
"workflowActions": [
  {
    "action": "approve",
    "label": "Approve",
    "targetStatus": "approved",
    "fromStatus": ["pending"],
    "confirm": {
      "message": "Approve this request?",
      "summary": ["request_no", "requested_by", "amount"]
    }
  }
]
```

| Property | Notes |
|---|---|
| `workflow.statusField` | Field name that holds the current status |
| `action` | Action identifier sent to backend `/change-status` |
| `label` | Button label |
| `targetStatus` | Status value after transition |
| `fromStatus[]` | Allowed current statuses for this action to appear |
| `confirm.message` | Confirmation dialog message |
| `confirm.summary[]` | Fields to display in the confirmation dialog |

---

## ID Generation

Supports auto-generated field values via `defaultValue.source: "idgen"`.
Requires `IDGEN_ENABLED=true` in backend config.

```json
{
  "name": "order_no",
  "type": "text",
  "defaultValue": {
    "source": "idgen",
    "mode": "serial",
    "prefix": "ORD",
    "format": "XXXX-XXXX"
  }
}
```

5 modes: `number` (sequential with prefix/format), `pin` (OTP numeric),
`code` (voucher-style), `serial` (license key pattern), `random`.
The field renders as readonly on the form; value is reserved on focus and
confirmed on submit.

---

## Live Sync

WebSocket-based live data refresh. Requires `LIVE_SYNC_ENABLED=true` and
`LIVE_SYNC_PORT` configured in backend.

Root-level config in UDF:
```json
"liveSync": {
  "url": "ws://localhost:3033",
  "apiKey": "your-api-key"
}
```

Per-page activation:
```json
"features": {
  "enableLiveSync": true
}
```

When active, the page's data table auto-refreshes when the backend pushes
an event over the WebSocket channel.

---

## Naming Conventions

| Element | Pattern | Example |
|---|---|---|
| `pageId` (CRUD) | `^[a-zA-Z0-9_-]+$` | `sales-order`, `product_list` |
| `pageId` (dashboard) | `^dash-[a-z0-9-]+$` | `dash-overview`, `dash-monthly` |
| `widgetId` | `^[a-z][a-z0-9-]*$` | `chart-revenue`, `kpi-orders` |
| field `name` | snake_case | `order_no`, `customer_id` |
| `appCode` | kebab-case | `sales-app`, `inventory` |
| `apiPath` | kebab-case with leading `/` | `"/sales-order"`, `"/product"` |
| `pageGroup` | Title Case | `"Master Data"`, `"Reports"` |

---

## Plugins

Built-in plugins. Run `designer_list_plugins` to confirm what the installed
version provides — do not hardcode this list.

| Plugin | Auth | Notes |
|---|---|---|
| `vanilla-js-basic` | None | Standard CRUD app, no login flow |
| `vanilla-js-auth` | Auth + RBAC | Login page, token refresh, role-based access |
| `vanilla-js-custom` | Auth + RBAC | Customisable markup/CSS/JS; auth + RBAC capable (confirm via `designer_list_plugins`) |

Plugin auth is built into the app at generation time. Disable it with
`noAuth: true` (`--no-auth`) on `designer_init_project` to get the plugin's UI
without auth. **Plugin auth (with RBAC) is distinct from the embedded `rfx_auth`
extension** (`designer_auth_create`, no RBAC) — see references/auth.md.

`vanilla-js-auth` accepts additional `appConfig` / init properties:
- `authAppCode` — must match the backend auth module's app code
- `idleTimeout` — auto-logout after inactivity (seconds)

**Custom plugins:**
Use `designer_scaffold_plugin` to generate a plugin template. The plugin
structure contains `plugin.json` (metadata) and Jinja2 templates for each
file type (HTML, JS, CSS). Use `designer_inspect_plugin` to verify capabilities
before referencing a plugin in a UDF payload.
