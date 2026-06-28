# Reference: Design-to-SDF Heuristics

> **Procedure, not a catalog dump.** This file is a classification procedure that
> produces a *draft* SDF. It does not replace `codegen_get_dbschema_catalog`:
> every draft is grounded against the catalog for valid options and MUST pass
> `codegen_dbschema_validate` before any DDL is generated. Inferences marked
> "confirm" are guesses — surface them to the user, never commit silently.

Use when the goal is to derive an SDF (Schema Definition File) table structure
from an external source. Exactly 5 source kinds are in scope — anything else
is out of scope and must be refused rather than guessed at:

| Source kind | Examples | Path |
|---|---|---|
| HTML | a UI mockup/template file | Inference path — Step 0-5 below |
| Image | screenshot, photo, pasted clipboard image (jpg/png) | Inference path — Step 0-5 below |
| JSON | sample API response, Figma API/plugin export | Direct-translation, low inference |
| Markdown | a written schema spec/data-dictionary document | Direct-translation, low inference |
| SQL DDL | `CREATE TABLE` statements/script, no live DB required | Direct-translation — see dedicated section below |

A "Figma export" is not its own category — it surfaces as either an Image
(screenshot/PNG export) or JSON (Figma API/plugin export), handled under
whichever of those two it actually is.

HTML and Image need **inference**: presentation must be classified into
stored/derived/snapshot/joined/relation/audit before any column is decided
(Step 0-5 below). JSON, Markdown, and SQL DDL are **already explicit** about
types and constraints — the work is mostly direct translation to catalog
syntax, with far less judgment-call risk. Do not run the Step 0-5
classification ceremony on these; map what is already stated.

Any source outside these 5 kinds (PDF requirements doc, verbal description,
ERD diagram tool export not covered above, etc.) is out of scope for this
reference — say so explicitly and ask how to proceed rather than improvising
a procedure for it.

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

## Step 0 — Confirm this is an SDF case at all

Not every design implies a new table. A **dashboard/analytics screen** maps to
the Dashboard RDF pipeline (`codegen_get_dashboard_catalog` →
`codegen_create_dashboard`, payload with `widgets` not `tableName`, page name
prefixed `dash-`), NOT to a new SDF table. Treating its numbers as columns
produces a fictitious table — e.g. a `dashboard` model with fields like
`sales`, `purchases`, `profit` is always wrong; those are query results, not
data ever written by an insert/update.

Signals that the source is a dashboard, not an entity to model:
- a global filter that parameterizes the whole view — a date range, a
  cross-entity dropdown (e.g. "Filter by warehouse") — rather than a filter
  scoped to one entity's list,
- stat cards / KPI tiles showing currency or count aggregates (Sales,
  Purchases, Profit, Total Due) with no per-card edit affordance,
  - a chart, especially a time-series one,
- the absence of all three Step 1 signals (no Add/Edit form, no per-entity
  filter, no row-level list) in their normal shape.

If any of these are present, stop before running Step 1-5. Instead:
1. Identify which **already-existing** tables the numbers aggregate from (here:
   `sale`, `purchase`, `sales_return`, `purchase_return`, `invoice`,
   `warehouse`). These need their own SDF only if they don't already exist —
   derived the normal way (via their own forms/lists), never from the
   dashboard screen itself.
2. Ground via `codegen_get_dashboard_catalog`, then build the dashboard payload
   with SQL widgets querying those tables, filtered by the date range /
   warehouse parameters shown.
3. Do not call `codegen_dbschema_init`/`migrate` for the dashboard itself —
   there is no table to create for it.

A design can mix both: a page with a dashboard section AND a genuine CRUD form
elsewhere. Apply this gate per-section, not to the whole file at once.

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

## SQL DDL source — direct translation path

Applies when the source is `CREATE TABLE` statements (a `.sql` file or a
pasted script), with or without a live database to run it against. This is
the lowest-inference path: the DDL already states every type and constraint
explicitly. Do not run Step 0-5 — translate what is written, syntax to
syntax, dialect to dialect-agnostic SDF.

### Type mapping

| SQL DDL type | SDF type |
|---|---|
| `VARCHAR(n)`, `CHARACTER VARYING(n)`, bounded `CHAR(n)` | `string:n` |
| `TEXT`, `CLOB`, unbounded `NVARCHAR` | `text` |
| `INT`, `INTEGER`, `INT4` | `integer` |
| `BIGINT`, `INT8` | `bigint` |
| `DECIMAL(p,s)`, `NUMERIC(p,s)` | `decimal:p,s` |
| `BOOLEAN`, `BOOL`, legacy `TINYINT(1)` flag | `boolean` |
| `DATE` | `date` |
| `TIMESTAMP`, `TIMESTAMPTZ`, `DATETIME` | `timestamp` |
| `UUID`, `UNIQUEIDENTIFIER` | `uuid` |
| `JSON`, `JSONB` | `json` |

### Constraint mapping

| SQL DDL | SDF |
|---|---|
| `PRIMARY KEY` | `pk` |
| `NOT NULL` | `notnull` |
| `UNIQUE` | `unique` |
| non-unique `INDEX`/`KEY` | `index` |
| `DEFAULT value` | `default:value` — same quoting rule as the catalog: string quoted, numeric/boolean raw, SQL constant bare, function e.g. `now()` |
| `CHECK (col IN (...))` | `checks: [{ field, in: [...] }]` |
| `CHECK (col =/!=/>/>=/</<= value)` | `checks: [{ field, eq/neq/gt/gte/lt/lte: value }]` |
| composite `PRIMARY KEY`/`UNIQUE` (multiple columns) | see `composite-primary-key.md` / `composite-unique.md` in the handbook catalog — do not improvise the syntax |

### Foreign keys — shorthand vs `relations`

Apply the same mutual-exclusivity rule as the catalog (`foreign-keys.md`):
- `FOREIGN KEY ... REFERENCES table(col)` with no `ON DELETE`/`ON UPDATE`
  clause (or only the dialect default) → `fk:table.col` shorthand.
- `FOREIGN KEY ... REFERENCES table(col) ON DELETE x ON UPDATE y` with an
  explicit, non-default action → a `relations` entry with that `onDelete`/
  `onUpdate`. Do NOT also write the `fk:` shorthand on the same field — the
  catalog rejects having both.
- The reference column is always the **actual column name** named in
  `REFERENCES table(column)` — this is given explicitly in DDL, so there is no
  `.id`-guessing risk here (unlike inferring FK targets from a UI dropdown).

### What NOT to add

Unlike the HTML/Image path, DDL is already a deliberate, complete spec from a
real system — do not apply Step 4's scaffolding defaults on top of it:
- Do **not** auto-add audit columns (`created_at`/`created_by`/`updated_at`/
  `updated_by`) if the DDL doesn't have them. Their absence is the source
  system's actual design, not a gap to fill.
- Do **not** convert the primary key strategy. If the DDL uses
  `SERIAL`/`AUTO_INCREMENT`/`GENERATED ALWAYS AS IDENTITY` (integer PK), keep
  `integer pk` — do not silently switch it to a `string:36` UUID PK because
  that happens to be the convention used elsewhere.

### Confirm-list additions specific to DDL

- If the DDL already has `is_deleted`/`deleted_at`/`deleted_by` columns, this
  is a soft-delete table — set `softDelete.enabled: true` and verify the
  target dialect is PostgreSQL (Phase 1 only); flag a conflict explicitly if
  the DDL's dialect is MySQL/Oracle/SQLite.
  - If the DDL's original dialect differs from the project's target dialect
  (e.g. migrating MySQL DDL into a PostgreSQL project), call out type/storage
  differences (e.g. MySQL boolean-as-`TINYINT(1)`) — the SDF type stays the
  dialect-agnostic logical type (`boolean`), the platform handles physical
  storage per target dialect.
- Named constraints in the source DDL (`pk_xxx`, `fk_xxx`) are not preserved —
  RESTForge generates its own deterministic constraint names on migrate. Note
  this only matters if something outside RESTForge depends on the original
  constraint names.

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

---

## Worked example — items.html (master-detail + lookup decision)

Design signals: Add Item form (Item Image *, Item Name *, Description *,
Price *, Net Price *, Category * dropdown, Tax * dropdown); two repeatable
accordion sections inside the same form — "Variations" (Size *, Price * per
row) and "Add Ons" (Name *, Price($) * per row); item cards in the list show
a Veg/Non Veg badge not present anywhere in the Add form.

This source exercises three rules the categories.html example doesn't:
FK reference correctness, the lookup-master-vs-enum decision on two fields
from the *same* form, and master-detail decomposition.

Classification:
- `Item Name`, `Description`, `Price`, `Net Price` → stored, required.
- `Item Image` → file upload → `string:255`.
- `Category *` → **FK to a master table**. Decisive signal: a dedicated
  `categories.html` admin page exists for it elsewhere in the same design set.
  → `category_id string:36 fk:category.category_id notnull` — the target
  column is `category.category_id` (the table's actual PK), not `category.id`.
- `Tax *` → **FK to a master table**, same reasoning — a dedicated
  `tax-settings.html` page exists. → `tax_id string:36 fk:tax.tax_id notnull`.
  The `tax` table itself is derived separately from its own admin page (title,
  rate, tax type), not invented here.
- `Veg / Non Veg` badge → **enum**, even though it appears only in the list,
  not the Add form. It's a small, permanent, intrinsic set with no admin page
  of its own (contrast with `Tax`, which looks similar — a fixed-looking
  label — but does have one). → `food_type string:20 default:'non_veg'` +
  `checks: [{ field: 'food_type', in: ['veg', 'non_veg'] }]`.
- "Variations" accordion (repeatable Size + Price rows) → **master-detail
  child table**, not columns on `item`. One row in the design = one row in a
  new `item_variation` table with `item_id` FK back to `item`.
- "Add Ons" accordion (repeatable Name + Price rows) → same pattern, a second
  child table `item_addon`.

Resulting draft SDF (four tables — `tax` is shown in outline; the master
entity and both detail tables in full):

```js
defineModel("tax", {
  fields: {
    tax_id:     "string:36 pk",
    title:      "string:100 notnull",
    rate:       "decimal:5,2 notnull",
    tax_type:   "string:20 notnull default:'exclusive'",
    created_at: "timestamp default:now()",
    created_by: "string:100",
    updated_at: "timestamp",
    updated_by: "string:100"
  },
  checks: [
    { field: "tax_type", in: ["inclusive", "exclusive"] }
  ]
});

defineModel("item", {
  fields: {
    item_id:     "string:36 pk",
    category_id: "string:36 fk:category.category_id notnull",
    item_name:   "string:150 notnull",
    description: "text notnull",
    item_image:  "string:255 notnull",
    food_type:   "string:20 notnull default:'non_veg'",
    price:       "decimal:10,2 notnull",
    net_price:   "decimal:10,2 notnull",
    tax_id:      "string:36 fk:tax.tax_id notnull",
    created_at:  "timestamp default:now()",
    created_by:  "string:100",
    updated_at:  "timestamp",
    updated_by:  "string:100"
  },
  checks: [
    { field: "food_type", in: ["veg", "non_veg"] }
  ]
});

defineModel("item_variation", {
  fields: {
    item_variation_id: "string:36 pk",
    item_id:           "string:36 fk:item.item_id notnull",
    size_name:         "string:50 notnull",
    price:             "decimal:10,2 notnull",
    created_at:        "timestamp default:now()",
    created_by:        "string:100",
    updated_at:        "timestamp",
    updated_by:        "string:100"
  }
});

defineModel("item_addon", {
  fields: {
    item_addon_id: "string:36 pk",
    item_id:       "string:36 fk:item.item_id notnull",
    addon_name:    "string:100 notnull",
    price:         "decimal:10,2 notnull",
    created_at:    "timestamp default:now()",
    created_by:    "string:100",
    updated_at:    "timestamp",
    updated_by:    "string:100"
  }
});
```

Confirm before migrate: `item_name`/`item_image` lengths, `food_type` default
value, whether `tax` needs more fields than the admin page showed, and that
both accordions are intentionally separate tables rather than columns on
`item`.

---

## Worked example — Order Details (no-form fallback + joined attribute + snapshot)

Design signals: a read-only "Order Details" page — **no Add/Edit form is
present in this source**. Panels: Order Info (Date Added, Payment Method,
Status), Customer Details (Name, Email, Phone, Country, State, Address),
Order Items (Product, Qty, Unit Price → line Total), summary (Subtotal, Tax,
Shipping Rate, Grand Total).

This source exercises three rules the previous two examples don't: the
no-form fallback, the joined-attribute exclusion, and the
derived-vs-snapshot distinction — including two fields (Address vs
Name/Email/Phone) that look identical in presentation but classify oppositely.

Classification:
- **No form in this source** → state that explicitly per the Step 1 fallback,
  proceed with this detail page as the best available signal rather than
  silently treating it as equivalent to a form.
- `Payment Method` → **enum**, not a master table. No admin/settings page or
  quick-add affordance is present anywhere in this source.
- `Status` → enum, closed permanent set (`pending`/`processing`/`completed`/
  `cancelled`).
- `Customer Name`, `Email`, `Phone` → **joined attribute**. These belong to
  `customer`, reachable through `customer_id`; not stored on `order`.
- `Country`, `State`, `Address` → **snapshot**, not joined — even though they
  look like the same kind of "customer info" as Name/Email/Phone just above
  them. The deciding question: could the customer's saved address change
  later while this order must still show where it actually shipped? Yes →
  must stay frozen on the order record.
- `Product`, `Qty` → relation (FK to `product`) + stored quantity, on a
  master-detail child table `order_item` (one row per line, same pattern as
  `item_variation` in the previous example).
- `Unit Price` (per line) and `Subtotal`/`Tax`/`Shipping Rate`/`Grand Total`
  (header summary) → **snapshot**. All are computed, but must not silently
  change if the product's current price changes after the order is placed.

Resulting draft SDF:

```js
defineModel("order", {
  fields: {
    order_id:         "string:36 pk",
    order_number:     "string:20 notnull unique",
    customer_id:      "string:36 fk:customer.customer_id notnull",
    payment_method:   "string:20 notnull",
    status:           "string:20 notnull default:'pending'",
    shipping_address: "text notnull",
    shipping_country: "string:100 notnull",
    shipping_state:   "string:100 notnull",
    subtotal:         "decimal:15,2 notnull",
    tax_amount:       "decimal:15,2 notnull default:0",
    shipping_rate:    "decimal:15,2 notnull default:0",
    grand_total:      "decimal:15,2 notnull",
    created_at:       "timestamp default:now()",
    created_by:       "string:100"
  },
  checks: [
    { field: "payment_method", in: ["credit_card", "bank_transfer", "cash", "e_wallet"] },
    { field: "status", in: ["pending", "processing", "completed", "cancelled"] }
  ]
});

defineModel("order_item", {
  fields: {
    order_item_id: "string:36 pk",
    order_id:      "string:36 fk:order.order_id notnull",
    product_id:    "string:36 fk:product.product_id notnull",
    qty:           "integer notnull",
    unit_price:    "decimal:15,2 notnull"
  }
});
```

Confirm before migrate: `payment_method` option set (this source gave no
admin-page signal — verify with the user whether the real system manages it
as a master table instead), `order_number` format/length, PK convention, and
that Name/Email/Phone were intentionally excluded as joined while
Address/totals were intentionally kept as snapshot — these two decisions look
similar on the screen but are opposite in the schema.
