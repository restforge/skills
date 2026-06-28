---
name: restforge
description: >
  RESTForge end-to-end workflow — use for any task involving RESTForge: SDF
  (Schema Definition File), RDF (Resource Definition File), UDF (UI Definition
  File), dbschema, defineModel, codegen, payload, generate endpoint, generate
  dashboard, generate frontend, migrate schema, setup project, add authentication
  (project auth backend, embedded rfx_auth frontend auth), or design-to-SDF
  (deriving a schema from an HTML mockup, screenshot, image, or UI design). Also
  active for the codegen_*, designer_*, setup_*, runtime_* MCP tools and the
  @restforgejs/platform / @restforgejs/mcp-server packages. Defines the canonical
  operation order for the backend track (setup → schema → payload → codegen →
  runtime) and the frontend track (init → udf → validate → generate), the
  grounding-first catalog rules, decision branches, and destructive-operation
  guardrails. Active whenever working on a RESTForge project, not only when the
  word "skill" is mentioned.
license: MIT
compatibility: >
  Requires the RESTForge MCP server (@restforgejs/mcp-server) registered in the
  client, plus a RESTForge license for codegen_*, runtime_*, and
  setup_validate_config operations. Designer tools do not require a license.
---

## Mental Model

RESTForge is a deterministic, definition-first generator with two output tracks:

- **Backend track** — SDF defines the database schema; RDF defines REST API
  endpoints. One SDF produces identical DDL; one RDF payload produces an
  identical endpoint module on every execution.
- **Frontend track** — UDF defines the frontend application. One UDF payload
  produces identical HTML/JS/CSS via `restforge-designer`, plugin-driven,
  no build step required.

The agent interacts with the platform **exclusively through MCP tools**. The
agent does not write generation code itself, does not modify generated output,
and does not guess options outside the catalog. All valid options come from the
catalog returned by grounding tools.

The platform exposes its capabilities as MCP tools grouped by domain: `health_*`,
`setup_*`, `codegen_*`, `runtime_*`, `designer_*`, `data_*`, `key_*`,
`project_*`. Two facts follow from this and shape every task:

1. **The skill describes intent and order; the MCP server executes.** If the
   RESTForge MCP server is not registered in the client, this skill cannot do
   anything — there is nothing to call. Confirm the tools are available before
   planning a multi-step operation.
2. **The catalog is the source of truth, not memory.** Field types, constraints,
   validation rules, and plugin capabilities belong to the *installed* platform
   version. Always ground against the catalog before proposing definition
   content (see Grounding-First Rules, below).

---

## Preflight (run before any RESTForge task)

Verify readiness before planning or producing anything:

1. **Are the RESTForge MCP tools present?** If `codegen_*` / `designer_*` are not
   in your available tools, the MCP server is **not active**. Stop and tell the
   user to register the RESTForge MCP server and restart the client. Do **not**
   finish the task by hand-reading the bundled `references/` — that bypasses the
   generator and yields slower, non-deterministic output.
2. **Backend work** → confirm the project and config are ready with
   `runtime_detect_project`, then the `setup_validate_config` gate.
3. **Frontend work** → the Designer tools pre-check that `restforge-designer` is
   on PATH; if it is missing, surface that before proceeding.

If a prerequisite is missing, report it as the next step — do not improvise around it.

---

## Backend Pipeline (canonical)

This is the canonical (golden) path. For state-dependent choices see Decision
Points; for failure handling see Guardrails and Common Errors.

The sequence below applies to a **new project from scratch**. For an existing
project, start from the step that matches the current state — do not re-run
earlier steps that already succeeded.

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
      Use setup_read_env to read existing values before overwriting.

5.  setup_validate_config
      ── GATE ── Must pass before any codegen_* operation starts.
      Validates database connection and license. Running codegen before this
      gate passes produces uninformative errors.

6.  codegen_get_dbschema_catalog
      ── GROUNDING ── Source of truth before defining SDF: field types,
      constraints, shorthand syntax, relations, referential actions,
      check operations, and the soft-delete contract.
      → references/dbschema-catalog.md

7a. codegen_dbschema_init         (empty DB — scaffold SDF with audit columns)
    codegen_dbschema_template     (minimal template, no DB connection required)
7b. codegen_dbschema_introspect   (existing DB — generate SDF from actual schema)

8.  codegen_dbschema_validate
      Validate SDF before any DDL is generated. Catch errors here, not at migrate.

9.  codegen_dbschema_generate_ddl  (optional — preview DDL without executing)

10a. codegen_dbschema_migrate      (empty DB — apply DDL directly)
10b. codegen_dbschema_diff         (existing DB — review the differences first)
     → codegen_dbschema_apply      (apply only after confirming the drift)

11. codegen_get_field_validation_catalog
      ── GROUNDING ── before defining fieldValidation in a payload.
      → references/field-validation.md

12. codegen_get_query_declarative_catalog
      ── GROUNDING ── when the payload contains datatablesQuery, viewQuery,
      viewName, exportQuery, or detailQuery.

13. codegen_generate_payload
      Generate payload JSON from a table. Foundation for all subsequent
      codegen operations.

14. codegen_validate_payload
      Validate the payload before codegen. Catch errors here.

15. codegen_diff_payload   (when a payload exists and the DB schema has changed)
    → codegen_sync_payload    (sync payload to the current DB state — non-breaking)
    → codegen_migrate_payload (when the payload has breaking changes)

16a. codegen_create_endpoint        (standard CRUD module)
16b. codegen_get_dashboard_catalog
     → codegen_create_dashboard     (analytic dashboard with SQL widgets)
16c. codegen_create_processor       (background processing)
16d. codegen_create_kafka_consumer  (Kafka event streaming)

17. runtime_generate_launcher
      Generate the launcher script. The agent STOPS here. The user executes
      the launcher — the server runs independently of the agent session.

18. (user executes the launcher)

19. runtime_check_status
      Verify the server is running and endpoints are reachable.
```

Two hard checkpoints (see Guardrails): **step 5** is a gate — no `codegen_*` call
before `setup_validate_config` passes; **step 17** is where the agent stops (it
generates the launcher, never runs the server).

---

## Frontend Pipeline (canonical)

This is the canonical (golden) path for the frontend track. It runs
**independently** from the backend pipeline. The backend API must be running and
reachable at `apiBaseUrl` before the generated frontend is useful, but the
frontend can be defined and generated without the backend live.

```
1.  designer_list_plugins
      ── GROUNDING ── list available output plugins before initializing.
      Built-in: vanilla-js-basic (no auth), vanilla-js-auth (JWT auth).
      → references/udf-catalog.md § Plugins

2.  designer_init_project
      Scaffold a new frontend project from a plugin. Creates the project folder,
      the initial UDF payload (payload.json), and plugin assets.

3.  designer_get_udf_catalog
      ── GROUNDING ── call before defining or editing any UDF payload.
      Returns valid field types, page anatomy, features, data-source formats,
      and validation rules for the installed plugin version.
      → references/udf-catalog.md

4.  [define / edit UDF payload JSON]
      Edit payload.json: appConfig, pages[], navigation[], homepage.
      One page entry = one CRUD page or one dashboard page.

5.  designer_validate_payload
      ── GATE ── validate the UDF payload. Catches structural errors before
      generation. Run before preview or generate, every time.

6.  designer_preview_files
      Dry-run: list files that would be generated, without writing to disk.
      Use to verify scope before an overwrite.

7.  designer_generate
      Generate frontend HTML/JS/CSS from the UDF payload. Writes output files.
      The agent STOPS here — the user opens the output in a browser.
```

For plugin development (custom output plugins):

```
designer_scaffold_plugin   →  [develop plugin templates]
  → designer_inspect_plugin    (verify plugin metadata and capabilities)
  → designer_generate          (test generation with the custom plugin)
```

Two checkpoints (see Guardrails): **step 5** is a gate — never `designer_generate`
an unvalidated payload; **step 7** is where the agent stops (generates files, does
not serve or deploy).

Licensing note: the Designer tools (`designer_preview_files`, `designer_generate`,
etc.) do **not** require a RESTForge license, unlike the `codegen_*`, `runtime_*`,
and `setup_validate_config` tools on the backend track.

---

## Auth Extension

RESTForge has **two different auth mechanisms — do not confuse them.** Pick the
right one; never describe one as the other, and never claim the extension does RBAC.

| Mechanism | RBAC? | How |
|---|---|---|
| **Plugin auth** — built into the frontend at generation time | **Yes (auth + RBAC)** | `vanilla-js-auth` or `vanilla-js-custom` plugin in `designer_init_project`; disable with `noAuth: true` (`--no-auth`) |
| **Auth extension** — bolt-on added to an existing project | **No RBAC** | backend `project_auth`; frontend `designer_auth_create` (embedded `rfx_auth`) |

Confirm which plugins provide auth with `designer_list_plugins` — do not hardcode
it. The rest of this section covers the **extension** (the no-RBAC bolt-on); for
plugin auth see Decision Points § Frontend plugin choice. Google Sign-In and
`@restforgejs/auth` are out of scope for the extension.

### Backend auth — `project_auth`

Adds the auth backend to an existing RESTForge project (run the standard backend
pipeline first; the project and its endpoint must already exist, and the DB must
be active).

```
project_auth   (wraps: npx restforge project auth --create --project=<name>)
```

Installs auth SDF (`rfx`), DB tables, middleware, router, six processors
(register/login/refresh/logout/me/reset-password), a random `JWT_SECRET`, and
`bcrypt`+`jsonwebtoken`. Idempotent. → references/auth.md § Backend

### Frontend auth — `designer_auth_create` / `designer_auth_remove`

Adds (or removes) an **embedded** login / signup / forget-password overlay
(`rfx_auth`) on an existing frontend project, at route
`/api/<project>/rfx_auth`. This is independent of the `vanilla-js-auth` plugin.

```
designer_auth_create   (wraps: restforge-designer auth --create --project=<name>)
designer_auth_remove   (wraps: restforge-designer auth --remove --project=<name> --force)
```

`create` writes the auth pages + `js/rfx_auth.js` and injects a guard into existing
pages; `remove` deletes them. Idempotent. `restforge-designer` must be on PATH.
→ references/auth.md § Frontend

Do not combine the two mechanisms on one app: if an app already has plugin auth
(`vanilla-js-auth` / `vanilla-js-custom`), do not also add embedded `rfx_auth`.

---

## Grounding-First Rules

Before reasoning about, proposing, or generating any **definition content** —
SDF fields, RDF `fieldValidation`, queries, dashboard widgets, or UDF pages —
call the matching grounding tool first and use only what it returns. Never write
definition content from memory.

| Context | Grounding tool | Reference |
|---|---|---|
| Defining or reviewing SDF | `codegen_get_dbschema_catalog` | references/dbschema-catalog.md |
| Deriving SDF from a UI design (HTML / image / screenshot) | `codegen_get_dbschema_catalog` | references/design-to-sdf.md |
| Defining `fieldValidation` in an RDF payload | `codegen_get_field_validation_catalog` | references/field-validation.md |
| Defining queries in an RDF payload | `codegen_get_query_declarative_catalog` | — |
| Defining a dashboard payload | `codegen_get_dashboard_catalog` | — |
| Defining UDF (frontend) | `designer_get_udf_catalog` | references/udf-catalog.md |
| Listing available frontend plugins | `designer_list_plugins` | references/udf-catalog.md § Plugins |

**Why this rule exists.** The catalog is the source of truth for valid options
in the *installed* platform version. Reasoning without it produces confident but
wrong output — field types that do not exist, constraints not applicable to a
type, or wrong semantics. Two concrete failure modes this rule prevents:

- **Inventing options.** Without grounding, an agent may write a made-up key
  (e.g. `"toUpper": true`) that the validator rejects. The catalog gives the
  exact key and value type.
- **Misreading semantics.** A constraint can mean something different from its
  plain-English name. For example, `uppercase` on a `string` field is a
  *normalization transform* (it forces the stored value to upper case), grouped
  with `trim` and `lowercase` — it is **not** a validator that rejects
  non-uppercase input. If the user wants rejection, the answer is `pattern`, not
  `uppercase`; if they want database-level enforcement, that is an SDF check
  constraint, not RDF `fieldValidation`. Grounding surfaces this distinction
  before the agent commits to the wrong one.

The reference files help you *understand* the catalog; they do **not** replace
the tool. Call the tool to ground, produce, and validate — do not hand-produce
output a tool would generate. The live tool is authoritative; when a reference and
the tool disagree, trust the tool.

---

## Decision Points

Use these to pick the correct branch when the request is state-dependent. Each
branch still obeys the Grounding-First Rules above.

### Schema (SDF)

- **DB does not exist** → `codegen_dbschema_init` (scaffold with audit columns)
  or `codegen_dbschema_template` (minimal template, no DB connection required).
- **DB already exists** → `codegen_dbschema_introspect` to generate SDF from the
  actual schema. Do not hand-write a schema that a real DB can describe.
- **Starting from a UI design** (HTML mockup, screenshot, image, Figma export)
  → first confirm the design is an entity to model, not a dashboard/analytics
  screen (those map to the Dashboard RDF branch below, not to a new SDF table).
  Then classify visible elements into stored / derived / relation / audit, draft
  the SDF, and run `codegen_dbschema_validate`. The design shows presentation,
  not storage — do not turn every visible label into a column.
  → references/design-to-sdf.md
- **DB exists with drift** → `codegen_dbschema_diff` to review, then
  `codegen_dbschema_apply`. Not `codegen_dbschema_migrate` — that is for empty
  DBs only.

### RDF Payload

- **No payload yet** → `codegen_generate_payload`.
- **Payload exists, edit is API-layer only** (e.g. add/adjust `fieldValidation`,
  messages, or a query — no column added or dropped) → ground via the matching
  catalog, edit `payload.json`, run `codegen_validate_payload`, then regenerate
  with `codegen_create_endpoint`. No DB migration: the schema (SDF) is unchanged.
- **Payload exists, DB schema changed** → `codegen_diff_payload` first, then
  `codegen_sync_payload` (non-breaking) or `codegen_migrate_payload` (breaking).
- **Custom query needed** → ground via `codegen_get_query_declarative_catalog`,
  then set `datatablesQuery`, `viewQuery`, or `exportQuery` in the payload.
  External SQL files use the `file:` prefix
  (e.g. `"datatablesQuery": "file:sql/orders.sql"`).

> **"Add an uppercase / case constraint" is ambiguous — disambiguate first.**
> In RESTForge `uppercase` is an RDF `fieldValidation` *normalization transform*
> (forces the stored value to upper case), not a reject-if-not-uppercase
> validator. If the user wants rejection, use `pattern`. If the user wants the
> database to enforce it, that is an SDF check constraint, not RDF. Confirm which
> one is meant before editing — see Grounding-First Rules § Misreading semantics.

### Backend module type

- **Standard CRUD** → `codegen_create_endpoint`. One payload = one module.
- **Analytic dashboard** → `codegen_create_dashboard`. Payload must have
  `widgets` (not `tableName`); page name must be prefixed `dash-`.
- **Background job** → `codegen_create_processor`.
- **Kafka event consumer** → `codegen_create_kafka_consumer`; requires
  `KAFKA_ENABLED=true` in config.
- **Workflow (status transitions)** → add `workflow` and `workflowActions` to the
  RDF payload; generates a `/change-status` endpoint automatically.
  → references/rdf-advanced.md § Workflow
- **Master-detail (composite CRUD)** → add `details[]` to the RDF payload;
  generates `/create-composite`, `/update-composite`, `/read-composite`.
  → references/rdf-advanced.md § Master-Detail
- **Excel export** → `/export` works by default (falls back to
  `SELECT {fields} FROM tableName`); customise the columns/filter with
  `exportQuery` in the RDF payload. Tune `EXPORT_FILE_EXPIRY` / `EXPORT_CHUNK_SIZE`
  in config. Ground `exportQuery` via `codegen_get_query_declarative_catalog`.
  → references/rdf-advanced.md § Data Source Resolution
- **Excel import (.xlsx)** → add `importConfig` (sheet, startRow, strategy,
  upsertKey, columns header→fieldName, optional lookup) to the RDF payload;
  generates `/import-preview` (validates, returns a diff) and `/import-commit`
  (applies). → references/rdf-advanced.md § Import Config
  - Activate on an existing project: edit the payload to add `importConfig`
    (and/or `exportQuery`) → `codegen_validate_payload` → `codegen_create_endpoint`.

### Soft-delete vs hard-delete

- Use soft-delete when deleted rows must be audited or recoverable. Declare
  `softDelete: { enabled: true }` in SDF and add the three contract columns
  (`is_deleted`, `deleted_at`, `deleted_by`).
- Soft-delete is supported on PostgreSQL only (Phase 1).
- Tables with composite UNIQUE constraints are incompatible with soft-delete.

### Data seeding / migration (rows, not schema)

Move table **rows** through SDF-driven envelope files. This is for data, never
for schema — use the dbschema tools for structure.

**Default output location:** `data-storage/<schema>/<table>.json`, relative to the
project cwd. The `data-storage` folder is the default of the `storagePath` param
(CLI `--storage-path <folder>`); override it to write elsewhere. The SDF read from
is `schemaPath` (CLI `--schema-path`, default `schema`).

- **Export / dump / snapshot / back up rows** → `data_pull`. Scope is exactly one
  of `table`, `schema`, or `allSchemas`. Only tables registered in the SDF can be
  pulled. `force: true` overwrites existing envelope files. Optional `limit`,
  `batchSize`, `config` (falls back to the default set via `config set-default`,
  i.e. `.restforge/defaults.json`), `schemaPath`, `storagePath`.
- **Import / load / seed / restore rows** → `data_push`. Same file names as
  `data_pull`, so pulled files push back directly. Scope is exactly one of `table`,
  `schema`, or `allSchemas`; for `schema`/`allSchemas` tables load in FK
  parent→child order.
- **Move data between databases** → `data_pull` from the source, then `data_push`
  into the target (`config` selects the env per side).
- ⚠ `data_push` is **APPEND-ONLY** (batch INSERT, no upsert/replace). Running it
  twice inserts the rows twice. Confirm with the user before pushing into a
  database that may already hold those rows.

### Frontend page type

- **Standard CRUD page** → `pageType: "crud"` (default) with `apiPath`,
  `primaryKey`, `displayField`, and `fields[]`.
- **Dashboard page** → `pageType: "dashboard"` with `dataSources[]` and `rows[]`
  containing widget columns.
  → references/udf-catalog.md § Dashboard Page
- **Page with approval workflow** → add `workflow.statusField` and
  `workflowActions[]` to the page.

### Frontend plugin choice

- **No auth** → `vanilla-js-basic`.
- **Auth WITH RBAC, built into the app** → `vanilla-js-auth` or
  `vanilla-js-custom` at `designer_init_project`. Use `noAuth: true` (`--no-auth`)
  to get the plugin's UI without its auth.
- **Custom branding / new plugin** → `designer_scaffold_plugin`.
- **Bolt-on auth WITHOUT RBAC** → not a plugin; see Authentication below.
  → references/udf-catalog.md § Plugins

### Authentication

- **Needs RBAC** → plugin auth (`vanilla-js-auth` / `vanilla-js-custom`) at
  `designer_init_project`. The extension below does **not** do RBAC.
- **Backend auth on an existing project (no RBAC)** → `project_auth` (after the
  project and its endpoint exist, with an active DB).
- **Frontend auth on an app built WITHOUT plugin auth (no RBAC)** →
  `designer_auth_create` (embedded `rfx_auth`).
- **Remove embedded frontend auth** → `designer_auth_remove` — destructive,
  confirm first (see Guardrails).

---

## Guardrails

These are hard rules. They override convenience and override an eager reading of
the user's request. When a guardrail conflicts with finishing faster, the
guardrail wins.

**1. Confirm before destructive operations.**
The following require explicit user confirmation before execution:

- `codegen_dbschema_migrate` or `codegen_dbschema_apply` that drops a table or
  column, or alters a column in a way that loses data.
- `project_delete` — permanent project deletion.

For schema changes, run `codegen_dbschema_diff` first, present a summary of the
destructive parts (dropped tables/columns, type narrowing), and wait for
confirmation before calling `codegen_dbschema_apply`. Never infer approval from
the original request — "update the schema" is not consent to drop a column.

**2. The agent does not run the server.**
The agent's last step on the backend track is `runtime_generate_launcher`. The
user executes the launcher in their own terminal. The agent never calls shell
commands to start, stop, or restart the server, and never assumes the server is
running — verify with `runtime_check_status` instead.

**3. The `validate_config` gate is mandatory.**
`setup_validate_config` must pass before any `codegen_*` operation starts. Do not
skip it "to save a step" — running codegen against an invalid config produces
uninformative errors that cost more time than the gate.

**4. Validate before generating.**
Always run `codegen_validate_payload` before `codegen_create_*`, and
`designer_validate_payload` before `designer_generate`. Generation on an invalid
payload produces incomplete or broken output that looks like it succeeded.

**5. Stay inside the task scope.**
Do not modify files outside the requested task. If the change is a backend
payload edit, do not also "tidy up" the SDF, the runtime, or the frontend. If a
change genuinely requires touching another area (e.g. an RDF edit that needs a
new column, which is an SDF change), stop and report the cross-over, then ask
before expanding scope.

**6. Ground before defining — repeated here because it is a guardrail, not a
suggestion.** Never write SDF fields, `fieldValidation`, queries, dashboard
widgets, or UDF content from memory. Call the matching catalog tool first (see
Grounding-First Rules). Inventing an option that "should" exist is the most
common way to produce confidently wrong output.

**7. Confirm before removing embedded auth.**
`designer_auth_remove` deletes the auth files (`login.html`, `signup.html`, the
forget-password overlay, `js/rfx_auth.js`) and strips the guard from existing
pages. Under MCP it runs with `--force` (no interactive prompt), so confirm the
project name and intent with the user **before** calling it.

**8. Execute through tools; never emulate them.** The bundled `references/` help
you *understand* options — they do not replace the tools. When a tool can produce
or validate an artifact (`codegen_dbschema_template`/`init`,
`codegen_generate_payload`, any `*_validate_*`), **call it**; do not hand-write its
output by reading a reference. Emulating the generator is slower, loses
determinism, and drifts from what the installed version emits. If the tool is not
available, stop (see Preflight) rather than improvising from the references.

---

## Prerequisites and Common Errors

### Environment prerequisites

- Node.js ≥ 18.
- A reachable database matching the configured `DB_TYPE` (PostgreSQL, MySQL,
  Oracle, or SQLite).
- A valid RESTForge license for `codegen_*`, `runtime_*`, and
  `setup_validate_config`. Designer tools do not require a license.
- The RESTForge MCP server registered in the client
  (`npm install -g @restforgejs/mcp-server`, then registered as the `restforge`
  MCP server). Without it, none of the tools below exist.
- Redis if using cache, distributed lock, or live sync.
- Kafka if using a Kafka consumer (`KAFKA_ENABLED=true`).

### Required backend config parameters

Nine of the full parameter set are mandatory before `setup_validate_config` can
pass:

`LICENSE`, `SERVER_ADDRESS`, `SERVER_PORT`, `DB_TYPE`, `DB_HOST`, `DB_PORT`,
`DB_USER`, `DB_PASSWORD`, `DB_NAME`.

→ references/config-schema.md for the full parameter list.

### Common error patterns

| Symptom | Cause | Recovery |
|---|---|---|
| Tool not found / no `codegen_*` tools available | MCP server not registered in the client | Install `@restforgejs/mcp-server`, register it as the `restforge` MCP server, restart the client |
| "authenticity check failed" / HTTP 401 | License not active or expired | Set `LICENSE` in `config/db-connection.env`, run `setup_validate_config` |
| HTTP 429 from license server | Rate limit (10 req/min/IP) | Expected since v5.1.15 — the client falls back to cache; wait or retry |
| DB connection failed | Wrong DB config or DB not running | Check `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`; verify the DB is running |
| "softDelete contract violation" | Contract columns missing or wrong type | Fix the SDF declaration, run `codegen_dbschema_validate` |
| "Widget uses undeclared placeholder" | `:paramName` in SQL not in `params` | Add the entry to `params` in the dashboard payload |
| "tableName and widgets conflict" | Payload contains both | Dashboard uses `widgets`; CRUD uses `tableName` — remove the wrong one |
| UDF validation error on field type | Field type not supported by the active plugin | Run `designer_list_plugins` + `designer_get_udf_catalog` to verify supported types |

When an error is not in this table, do not guess a fix. Re-run the relevant
`*_validate_*` tool, read its message, and ground against the catalog before
changing the definition.
