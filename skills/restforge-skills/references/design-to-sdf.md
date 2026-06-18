# Reference: Design-to-SDF Heuristics

Use when the input is a UI design — an HTML mockup, a screenshot/image, or a
Figma export — and the goal is to derive an SDF (Schema Definition File) table
structure from it.

This reference does NOT replace the catalog. It is a classification procedure
that produces a **draft** SDF. The draft is always grounded against
`codegen_get_dbschema_catalog` for valid options and MUST pass
`codegen_dbschema_validate` before any DDL is generated. Inferences marked
"confirm" below are guesses — surface them to the user, do not silently commit.

---

## Why a procedure is needed

A design shows *presentation*, not *storage*. The same screen mixes four kinds
of things that map very differently to a schema:

| Kind | Example in design | SDF treatment |
|---|---|---|
| Stored column | "Category Name" text input | a field in `fields` |
| Derived value | "No of Items" count in a table | NOT a column — computed via relation/query |
| Relation | "Category" dropdown on an Items form | FK + `belongsTo` relation |
| Audit/system | "Created On", "Last Updated" | audit columns, not new fields |

Reading every visible label as a column is the most common error. Classify
first, then assign types.

---

## Step 1 — Locate the authoritative element

Prefer signals in this order. Earlier signals are more reliable than later.

1. **Add/Edit form (modal or page)** — the create/update form is the single
   best source of *stored* columns. Every editable input is a candidate column.
2. **Filter panel** — reveals enumerable values (status options) and filterable
   relations (category dropdown).
3. **Table/list `<thead>`** — confirms which columns are *displayed*, but mixes
   stored, derived, and relation-label columns. Use to cross-check, not as the
   primary source.

A field that appears only in the table but never in the form is usually
**derived** or a **relation label**, not a stored column.

---

## Step 2 — Classify each candidate

For every input found in the form, decide its kind before its type.

- **Stored column** — free input the user types/picks and that belongs to *this*
  entity. → goes in `fields`.
- **Relation (FK)** — a dropdown/autocomplete that selects *another* entity
  (e.g. an Items form with a "Category" select). The stored column is the
  foreign key (`category_id`), and a `belongsTo` relation is added. The visible
  label ("Category name") is NOT stored on this table.
  - The FK reference uses the **actual PK column name** of the target table:
    `fk:category.category_id`, NOT `fk:category.id`. RESTForge does not resolve
    `.id` to the primary key — a non-existent target column is rejected.
  - A `*` on the dropdown means the relation is mandatory → add `notnull` to the
    FK column. Do NOT downgrade a required relation to a nullable free-text
    label (e.g. modelling a required "Tax" select as `tax_label string:50` drops
    both the relation and the constraint — wrong on two counts).
  - If a target entity for the dropdown plausibly exists (its own admin/settings
    page, e.g. Tax → tax-settings), model it as an FK. Only fall back to an enum
    `checks` set when the options are a small fixed list with no backing entity.
- **Enum** — a select/radio whose options are a small, closed, *permanent* set
  intrinsic to the data (e.g. Gender: Male / Female; Status: Active / Inactive;
  Tax Type: Inclusive / Exclusive). → stored as `string` + a `checks` entry with
  `in: [...]`. No separate table.
- **Lookup master** — a dropdown whose options are *managed data* that can grow,
  shrink, or change over time, or that carry extra attributes (a rate, an image,
  a description). → model as a separate **master table** and an FK, never as an
  enum. See the decision rule below.
- **Derived** — counts, sums, totals, "No of Items", "Total Orders". → NOT a
  column. Note it for an aggregate query (RDF `viewQuery`/`datatablesQuery`) or
  a `hasMany` relation count, and exclude from `fields`.
- **Audit/system** — "Created On/At", "Updated On/At", "Created By". → map to
  the standard audit columns, do not invent new fields.

### Decision rule — lookup master table vs enum

When a field offers a fixed set of choices, decide between a **master table + FK**
and an inline **enum (`checks`)** by permanence and management:

- **Non-permanent / managed lookup → ALWAYS a master table + FK.** If the option
  set is business data that an admin adds to, edits, deactivates, or that carries
  extra attributes, it is a managed entity — model it as its own table and
  reference it by FK. Examples: Category, Brand, Tax, Unit, Warehouse, City,
  Payment Method.
- **Simple, permanent, closed set → enum (`checks in:[...]`).** If the values are
  intrinsic to the record, will not be managed through a UI, and the set is
  stable, store a `string` with a `checks` constraint. Examples: Gender
  (`MALE`/`FEMALE`), Active/Inactive status, Yes/No, Tax Type
  (Inclusive/Exclusive).

Decisive signals for a master table (any one is sufficient):
- a dedicated admin/settings page exists for it (e.g. Tax → `tax-settings`,
  Category → `categories`),
- options carry their own attributes (rate, image, description, code),
- the business is expected to add/remove options over time.

When uncertain, prefer the **master table** — promoting an enum to a table later
is a breaking migration, while a stable enum rarely needs to become a table.
Note: a master table can itself contain a permanent enum column (e.g. the `tax`
table stores `tax_type` as an enum).

---

## Step 3 — Assign SDF types

Map the HTML control (or the visual control in an image) to a catalog field
type. All types below are from `codegen_get_dbschema_catalog`.

| Design control | SDF type | Notes |
|---|---|---|
| Short text input | `string:N` | N = confirm; default 255 if no hint |
| Long text / `<textarea>` | `text` | description, notes, address |
| Number (count, qty) | `integer` / `bigint` | bigint only if very large |
| Money / price | `decimal:15,2` | currency; scale 2 unless design says otherwise |
| Percentage / rate | `decimal:5,2` | confirm precision |
| Single checkbox / toggle | `boolean` | true/false only |
| Status / type select | `string:20` + `checks in:[...]` | enum, not boolean, when >2 states |
| Date picker | `date` | date only |
| Date + time | `timestamp` | |
| File / image upload | `string:500` | store the path/URL, never the binary |
| Email | `string:255` | |
| Phone | `string:30` | |
| Color picker / code | `string:7` or `string:36` | hex vs token |
| Entity dropdown (FK) | `string:36` or `integer` + `fk:<table>.<col>` | match the target PK type |
| JSON / key-value block | `json` | |

Rules carried from the catalog:
- `string` always needs a length; `decimal` always needs precision,scale.
- A `*` / "required" marker → add `notnull`.
- A select with fixed options → `checks: [{ field, in: [...] }]`, options
  lowercased/normalized to storage values.
- FK uses dot notation `fk:<table>.<column>`; also add the `belongsTo` relation
  with `onDelete` (default `restrict`; confirm).

---

## Step 4 — Add structural scaffolding

A design rarely shows these, but a RESTForge table needs them:

- **Primary key** — add `<entity>_id` as `string:36 pk` (UUID style) or
  `integer pk` (serial style). Confirm the convention with the user/project.
- **Audit columns** — add the four standard audit columns
  (`created_at`, `created_by`, `updated_at`, `updated_by`) unless this is a pure
  lookup table. "Created On" in the design maps here.
- **Soft-delete** — only if the design shows a recycle bin / "restore" / archive
  affordance. Then apply the soft-delete contract (PostgreSQL only).

---

## Step 5 — Validate and confirm

1. Write the draft SDF (`defineModel`) using only catalog options.
2. Run `codegen_dbschema_validate`.
3. Present the **confirm list** to the user before migrating:
   - every guessed `string` length and `decimal` precision,
   - every enum option set,
   - every FK target and its `onDelete`,
   - PK convention (uuid vs serial),
   - any "derived" column excluded from the schema (state it explicitly so the
     omission is intentional, not silent).

Never call `codegen_dbschema_migrate`/`apply` on a design-derived SDF without
this confirmation.

---

## Worked example — categories.html

Design signals: table columns (Category, No of Items, Created On, Status,
Actions); Add/Edit form (Category Image *, Category Name *); filter (Status:
Active / Inactive).

Classification:
- `Category Name` → stored, required → `string:255 notnull`.
- `Category Image` → file upload → `string:500` (path/URL).
- `Status` → enum (Active/Inactive) → `string:20 default:'active'` + check.
- `No of Items` → **derived** (count of `items` with FK to category) → excluded
  from `fields`; expose later via aggregate query or `hasMany` count.
- `Created On` → audit → `created_at`.
- `Actions` → UI only → ignored.

Resulting draft SDF:

```js
defineModel("category", {
  fields: {
    category_id:   "string:36 pk",
    category_name: "string:255 notnull",
    image_url:     "string:500",
    status:        "string:20 notnull default:'active'",
    created_at:    "timestamp default:now()",
    created_by:    "string:100",
    updated_at:    "timestamp",
    updated_by:    "string:100"
  },
  checks: [
    { field: "status", in: ["active", "inactive"] }
  ]
})
```

Confirm before migrate: `category_name` length (255?), `status` option values,
PK convention (uuid vs serial), and that `No of Items` is intentionally derived,
not stored.
