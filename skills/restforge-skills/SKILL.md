---
name: restforge
description: >
  RESTForge end-to-end workflow skill — use for every task involving RESTForge,
  SDF (Schema Definition File), RDF (Resource Definition File), UDF (UI
  Definition File), dbschema, defineModel, codegen, payload, generate endpoint,
  generate dashboard, generate frontend, migrate schema, setup project,
  design-to-SDF, deriving a schema from an HTML mockup / screenshot / image / UI
  design, codegen_*, designer_*, restforge-designer, @restforgejs/platform, or
  @restforgejs/mcp-server. This skill defines the canonical operation sequence
  for both backend (setup → schema → payload → codegen → runtime) and frontend
  (init → udf → validate → generate), catalog grounding rules, decision
  branches, and safety guardrails. Active whenever working on a RESTForge
  project — not only when the word "skill" is mentioned.
---

## Mental Model

RESTForge is a deterministic, definition-first generator with two output tracks:

- **Backend track** — SDF defines the database schema; RDF defines REST API
  endpoints. One SDF produces identical DDL; one RDF payload produces an
  identical endpoint module on every execution.
- **Frontend track** — UDF defines the frontend application. One UDF payload
  produces identical HTML/JS/CSS via `restforge-designer`, plugin-driven,
  no build step required.

The agent interacts with the platform exclusively through MCP tools. The agent
does not write generation code itself, does not modify generated output, and
does not guess options outside the catalog. All valid options come from the
catalog returned by grounding tools.

The platform exposes 61 tools across domains: `health_*`, `setup_*`,
`codegen_*`, `runtime_*`, `designer_*`, `data_*`, `key_*`, `project_*`.

---

## Backend Pipeline (canonical)

The sequence below applies to a new project from scratch. For an existing
project, start from the step that matches the current state.

```
1.  setup_create_folder
      Create a new project folder at the specified location.

2.  setup_install_package
      Install @restforgejs/platform into the project folder.

3.  setup_init_config
      Write config/db-connection.env from the default template.

4.  setup_write_env  /  setup_update_env
      Set values: LICENSE, SERVER_ADDRESS, SERVER_PORT, DB_TYPE,
      DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME (9 required parameters).
      Use setup_read_env to read existing values.

5.  setup_validate_config
      ── GATE ── Must pass before any codegen_* operation starts.
      Validates database connection and license. Running codegen before this
      gate passes produces uninformative errors.

6.  codegen_get_dbschema_catalog
      Grounding before defining SDF. Source of truth for field types,
      constraints, shorthand syntax, relations, referential actions,
      check operations, and soft-delete contract.
      → See references/dbschema-catalog.md for catalog content.

7a. codegen_dbschema_init         (empty DB — scaffold SDF with audit columns)
    codegen_dbschema_template     (minimal template, no DB connection required)
7b. codegen_dbschema_introspect   (existing DB — generate SDF from actual schema)

8.  codegen_dbschema_validate
      Validate SDF before DDL is generated. Catch errors here, not at migrate.

9.  codegen_dbschema_generate_ddl  (optional — preview DDL without executing)

10a. codegen_dbschema_migrate      (empty DB — apply DDL directly)
10b. codegen_dbschema_diff         (existing DB — review differences first)
     → codegen_dbschema_apply      (apply after confirming drift)

11. codegen_get_field_validation_catalog
      Grounding before defining fieldValidation in a payload.
      → See references/field-validation.md for catalog content.

12. codegen_get_query_declarative_catalog
      Grounding when payload contains datatablesQuery, viewQuery, viewName,
      exportQuery, or detailQuery.

13. codegen_generate_payload
      Generate payload JSON from a table. Foundation for all subsequent
      codegen operations.

14. codegen_validate_payload
      Validate payload before codegen. Catch errors here.

15. codegen_diff_payload  (when payload exists and DB schema has changed)
    → codegen_sync_payload  (sync payload to current DB state)
    → codegen_migrate_payload  (when payload has breaking changes)

16a. codegen_create_endpoint        (standard CRUD module)
16b. codegen_get_dashboard_catalog
     → codegen_create_dashboard     (analytic dashboard with SQL widgets)
16c. codegen_create_processor       (background processing)
16d. codegen_create_kafka_consumer  (Kafka event streaming)

17. runtime_generate_launcher
      Generate the launcher script. The agent stops here. The user executes
      the launcher — the server runs independently of the AI session.

18. (user executes launcher)

19. runtime_check_status
      Verify the server is running and endpoints are reachable.
```

---

## Frontend Pipeline (canonical)

The frontend pipeline runs independently from the backend pipeline. The backend
API must be running and reachable at `apiBaseUrl` before the frontend is useful.

```
1.  designer_list_plugins
      Grounding — list available output plugins before initializing a project.
      Built-in: vanilla-js-basic (no auth), vanilla-js-auth (JWT auth).
      → See references/udf-catalog.md § Plugins.

2.  designer_init_project
      Scaffold a new frontend project from a plugin. Creates the project folder,
      initial UDF payload (payload.json), and plugin assets.

3.  designer_get_udf_catalog
      ── GROUNDING ── Call before defining or editing any UDF payload.
      Returns valid field types, page anatomy, features, data source formats,
      and validation rules for the installed plugin version.
      → See references/udf-catalog.md for catalog content.

4.  [define / edit UDF payload JSON]
      Edit payload.json: appConfig, pages[], navigation[], homepage.
      One page entry = one CRUD page or one dashboard page.

5.  designer_validate_payload
      Validate UDF payload. Catches structural errors before generation —
      run this before preview or generate.

6.  designer_preview_files
      Dry-run: list files that would be generated without writing to disk.
      Use to verify scope before overwrite.

7.  designer_generate
      Generate frontend HTML/JS/CSS from UDF payload. Writes output files.
      Agent stops here — user opens the output in a browser.
```

For plugin development (custom output plugins):
```
designer_scaffold_plugin  →  [develop plugin templates]
  → designer_inspect_plugin  (verify plugin metadata and capabilities)
  → designer_generate        (test generation with custom plugin)
```

---

## Grounding-First Rules

Before reasoning, proposing, or generating any definition content — call the
relevant grounding catalog first.

| Context | Grounding tool | Reference |
|---|---|---|
| Defining or reviewing SDF | `codegen_get_dbschema_catalog` | references/dbschema-catalog.md |
| Deriving SDF from a UI design (HTML/image) | `codegen_get_dbschema_catalog` | references/design-to-sdf.md |
| Defining `fieldValidation` in RDF payload | `codegen_get_field_validation_catalog` | references/field-validation.md |
| Defining queries in RDF payload | `codegen_get_query_declarative_catalog` | — |
| Defining dashboard payload | `codegen_get_dashboard_catalog` | — |
| Defining UDF (frontend) | `designer_get_udf_catalog` | references/udf-catalog.md |
| Listing available frontend plugins | `designer_list_plugins` | references/udf-catalog.md § Plugins |

Reason: the catalog is the source of truth for valid options in the installed
platform version. Reasoning without the catalog produces incorrect assumptions —
field types that do not exist, invalid constraints, or wrong ordering.

---

## Decision Points

### Schema (SDF)

- **DB does not exist** → `codegen_dbschema_init` (scaffold with audit columns)
  or `codegen_dbschema_template` (minimal template).
- **DB already exists** → `codegen_dbschema_introspect` to generate SDF from
  the actual schema.
- **Starting from a UI design** (HTML mockup, screenshot, image, Figma export)
  → first confirm the design is an entity to model, not a dashboard/analytics
  screen (those map to the Dashboard RDF pipeline below, not to a new SDF
  table). Then classify into stored / derived / relation / audit elements,
  produce a draft SDF, then `codegen_dbschema_validate`. The design shows
  presentation, not storage — do not read every visible label as a column.
  → See references/design-to-sdf.md for the classification procedure.
- **DB exists with drift** → `codegen_dbschema_diff` to review, then
  `codegen_dbschema_apply`. Not `codegen_dbschema_migrate` — that is for empty DBs only.

### RDF Payload

- **No payload yet** → `codegen_generate_payload`.
- **Payload exists, DB changed** → `codegen_diff_payload` first, then
  `codegen_sync_payload` (non-breaking) or `codegen_migrate_payload` (breaking).
- **Custom query needed** → ground via `codegen_get_query_declarative_catalog`,
  then set `datatablesQuery`, `viewQuery`, or `exportQuery` in the payload.
  External SQL files use `file:` prefix (e.g., `"datatablesQuery": "file:sql/orders.sql"`).

### Backend module type

- **Standard CRUD** → `codegen_create_endpoint`. One payload = one module.
- **Analytic dashboard** → `codegen_create_dashboard`. Payload must have
  `widgets` (not `tableName`); page name must be prefixed `dash-`.
- **Background job** → `codegen_create_processor`.
- **Kafka event consumer** → `codegen_create_kafka_consumer`; requires
  `KAFKA_ENABLED=true` in config.
- **Workflow (status transitions)** → add `workflow` and `workflowActions`
  to RDF payload; generates `/change-status` endpoint automatically.
  → See references/rdf-advanced.md § Workflow.
- **Master-detail (composite CRUD)** → add `details[]` to RDF payload;
  generates `/create-composite`, `/update-composite`, `/read-composite`.
  → See references/rdf-advanced.md § Master-Detail.

### Soft-delete vs hard-delete

- Use soft-delete when deleted rows must be audited or recoverable. Declare
  `softDelete: { enabled: true }` in SDF and add three contract columns
  (`is_deleted`, `deleted_at`, `deleted_by`).
- Soft-delete is only supported on PostgreSQL (Phase 1).
- Tables with composite UNIQUE constraints are incompatible with soft-delete.

### Frontend page type

- **Standard CRUD page** → set `pageType: "crud"` (default) with `apiPath`,
  `primaryKey`, `displayField`, and `fields[]`.
- **Dashboard page** → set `pageType: "dashboard"` with `dataSources[]`
  and `rows[]` containing widget columns.
  → See references/udf-catalog.md § Dashboard Page.
- **Page with approval workflow** → add `workflow.statusField` and
  `workflowActions[]` to the page.

### Frontend plugin choice

- **No authentication required** → use `vanilla-js-basic`.
- **JWT authentication required** → use `vanilla-js-auth`.
- **Custom branding / behavior** → `designer_scaffold_plugin` to create a
  new plugin from template.
  → See references/udf-catalog.md § Plugins.

---

## Guardrails

**1. Confirm before destructive operations.**
The following must be confirmed by the user before executing:
- `codegen_dbschema_migrate` or `codegen_dbschema_apply` that drops tables
  or columns.
- `project_delete` — permanent project deletion.

Present a summary of destructive changes from `codegen_dbschema_diff` and
wait for confirmation before calling `codegen_dbschema_apply`.

**2. The agent does not run the server.**
The agent only generates a launcher via `runtime_generate_launcher`. The user
executes the launcher in their terminal. The agent never calls shell commands
to start, stop, or restart the server.

**3. validate_config gate.**
`setup_validate_config` must pass before any `codegen_*` operation starts.
Do not skip this gate for any reason.

**4. designer_validate_payload before generate.**
Always run `designer_validate_payload` before `designer_generate`. Generation
on an invalid UDF payload produces incomplete or broken output.

**5. Task scope.**
Do not modify files outside the scope of the requested task. If changes touch
another component, report and ask for confirmation instead of silently
expanding scope.

---

## Prerequisites and Common Errors

### Environment prerequisites

- Node.js ≥ 18
- Active and reachable database matching the configured `DB_TYPE`
- Valid RESTForge license key for `codegen_*`, `runtime_*`, and `setup_validate_config`
  (Designer tools — `designer_preview_files`, `designer_generate`, etc. — do not require a license)
- `@restforgejs/mcp-server` installed globally (`npm install -g @restforgejs/mcp-server`)
- Active Redis if using cache, distributed lock, or live sync
- Active Kafka if using Kafka consumer (`KAFKA_ENABLED=true`)

### Required backend config parameters (9 of 63)

`LICENSE`, `SERVER_ADDRESS`, `SERVER_PORT`, `DB_TYPE`, `DB_HOST`, `DB_PORT`,
`DB_USER`, `DB_PASSWORD`, `DB_NAME`.

For the full list of 63 parameters, see `references/config-schema.md`.

### Common error patterns

| Error | Cause | Recovery |
|---|---|---|
| "authenticity check failed" / HTTP 401 | License not active or expired | Set `LICENSE` in `config/db-connection.env`, run `setup_validate_config` |
| HTTP 429 from license server | Rate limit (10 req/min/IP) | Normal since v5.1.15 — client falls back to cache, wait or retry |
| DB connection failed | Wrong DB config or DB not running | Check DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME; verify DB is running |
| "softDelete contract violation" | Contract columns missing or wrong type | Fix SDF declaration, run `codegen_dbschema_validate` |
| "Widget uses undeclared placeholder" | `:paramName` in SQL not in `params` | Add entry to `params` in dashboard payload |
| "tableName and widgets conflict" | Payload contains both | Dashboard uses `widgets`; CRUD uses `tableName` |
| UDF validation error on field type | Field type not supported by active plugin | Run `designer_list_plugins` + `designer_get_udf_catalog` to verify supported types |
