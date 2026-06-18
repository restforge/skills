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
| Snapshot value | "Unit Price", "Grand Total" on an order | a field in `fields` — looks computed but must be frozen |
| Joined attribute | "Customer Email" on an order detail page | NOT a column on this table — belongs to the related entity |
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

### When the given source has no form

A read-only detail/view page (e.g. an order "details" screen with no `<input>`)
is NOT an authoritative source by itself — it is a list/table-equivalent (signal
3), not signal 1. Before classifying from it directly:

- Look for a sibling Add/Edit page or modal. List and detail pages commonly link
  to it (e.g. an "Edit" button pointing to `edit-order.html` / `add-order.html`).
  If found, treat that form as the authoritative source for Step 2 and use the
  detail page only to cross-check.
- If no Add/Edit form exists anywhere in the provided source(s), say so
  explicitly and proceed with the detail page as the best available signal —
  do not silently treat it as equivalent to a form.

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
- **Joined attribute** — a value displayed on this screen but that is actually
  a column of a *related* entity, shown via join (e.g. "Customer Email" /
  "Customer Phone" on an order detail page — these belong to `customer`, not
  `order`). → NOT a column on this table. Confirm: does this entity already have
  its own table where the field belongs? If the related entity's FK is already
  modelled (e.g. `customer_id`), the attribute is reachable through that
  relation and must not be duplicated here.
- **Derived vs. Snapshot** — both look like "a number that's computed", but they
  must be treated oppositely. Ask: *if the source data changes later, must this
  value stay frozen on this record, or is it allowed to change?*
  - **Derived** (allowed to change, recomputed live) — counts, running totals,
    "No of Items", "Total Orders". → NOT a column. Note it for an aggregate
    query (RDF `viewQuery`/`datatablesQuery`) or a `hasMany` relation count, and
    exclude from `fields`.
  - **Snapshot** (must stay frozen, computed or copied once at the moment of the
    transaction) — IS a column, even though it looks computed or looks like it
    duplicates a related entity's field. The principle is not limited to money:
    anything copied from a related entity onto a transaction record, where that
    related entity could change *after* the transaction while the transaction
    must keep showing the original value at the time. Excluding it from
    `fields` lets historical records change retroactively when the source
    changes — a data-integrity bug, not a simplification. Recognize it by
    asking "could the source of this value be edited or deleted later, while
    this record must still show what was true at transaction time?" — examples:
    - money: line `unit_price`, `subtotal`, `tax_amount`, `shipping_rate`,
      `grand_total` on an order; a price/rate copied from a master table.
    - identity/labels: a product's `name`/`sku` copied onto an order line (the
      product can be renamed or discontinued later; the historical order line
      must still show what was ordered).
    - location: a `shipping_address`/`billing_address` copied onto an order
      (the customer's address book entry can change later; the order must keep
      shipping to where it said it would at the time of purchase).
  - This overlaps with **Joined attribute** above — both involve a value that
    *could* be reached via a relation. The deciding question is whether the
    value is allowed to drift when the related entity changes (joined — do not
    store) or must stay frozen (snapshot — store). When the field is tied to a
    specific transaction/sale and could plausibly change at the source later
    (price, product label, address), default to **snapshot** — under-storing
    breaks historical accuracy; over-storing only costs a redundant column.
    Pure identity/contact fields with no transactional freezing need (e.g. a
    customer's current email shown on their own profile page) stay **joined**.
- **Audit/system** — "Created On/At", "Updated On/At", "Created By". → map to
  the standard audit columns, do not invent new fields.

### Decision rule — lookup master table vs enum

When a field offers a fixed set of choices, decide between a **master table + FK**
and an inline **enum (`checks`)** by permanence and management:

- **Non-permanent / managed lookup → ALWAYS a master table + FK.** If the option
  set is business data that an admin adds to, edits, deactivates, or that carries
  extra attributes, it is a managed entity — model it as its own table and
  reference it by FK. Examples: Category, Brand, Tax, Unit, Warehouse, City.
- **Simple, permanent, closed set → enum (`checks in:[...]`).** If the values are
  intrinsic to the record, will not be managed through a UI, and the set is
  stable, store a `string` with a `checks` constraint. Examples: Gender
  (`MALE`/`FEMALE`), Active/Inactive status, Yes/No, Tax Type
  (Inclusive/Exclusive).

These examples are illustrative, not fixed labels — the same field name can be
either depending on the actual design. Decide every case by the signals below,
not by matching the field's name against an example list.

Decisive signals for a master table (any one is sufficient):
- a dedicated admin/settings page exists for it (e.g. Tax → `tax-settings`,
  Category → `categories`),
- options carry their own attributes (rate, image, description, code),
- the business is expected to add/remove options over time,
- a quick-add affordance sits directly on the dropdown in the same form (e.g. a
  "+" button next to "Choose Category" / "Choose Brand") — this is direct UI
  evidence of a managed entity and needs no separate admin page to confirm it.

The absence of a signal is not evidence for the opposite conclusion. In
particular, a dropdown with no "+" button is NOT thereby an enum — check the
other signals before concluding enum. A dropdown whose options are themselves
children of an already-managed entity (e.g. "Subcategories" depending on a
selected "Category") is still a master-table FK even with no quick-add button
of its own, because it inherits "managed" status from its parent entity.

Without any of these signals present in the provided source(s), do not assume
a master table — a label by itself (e.g. "Payment Method: Credit Card" on an
order detail page, with no payment-method admin page or quick-add affordance in
evidence) defaults to an enum. "Payment Method" in particular varies by system:
a simple storefront with a fixed set of methods is an enum; a system with a
payment-gateway settings page (fees, codes, enable/disable per method) is a
master table. Confirm with the user when the source gives no signal either way.

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
