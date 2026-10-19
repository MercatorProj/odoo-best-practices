# ORM Patterns

## Model Definition

```python
from odoo import api, fields, models
from odoo.exceptions import ValidationError


class MyModel(models.Model):
    _name = 'my.model'
    _description = 'My Model'
    _order = 'sequence, name'

    name = fields.Char(required=True, index=True)
    sequence = fields.Integer(default=10)
    state = fields.Selection([
        ('draft', 'Draft'),
        ('confirmed', 'Confirmed'),
        ('done', 'Done'),
    ], default='draft', tracking=True)
    partner_id = fields.Many2one('res.partner', string='Partner', index=True)
    line_ids = fields.One2many('my.model.line', 'order_id', string='Lines')
    amount_total = fields.Monetary(compute='_compute_amount_total', store=True)
    currency_id = fields.Many2one(related='company_id.currency_id', readonly=True)
    company_id = fields.Many2one('res.company', default=lambda self: self.env.company)
    active = fields.Boolean(default=True)
    ref = fields.Char(copy=False, readonly=True)
```

## Computed Fields

Always declare `compute`, and add `store=True` only when the value is needed in searches or group-bys. Add `@api.depends` for every field the computation reads.

```python
amount_total = fields.Monetary(compute='_compute_amount_total', store=True)

@api.depends('line_ids.amount_subtotal')
def _compute_amount_total(self):
    for rec in self:
        rec.amount_total = sum(rec.line_ids.mapped('amount_subtotal'))
```

For inverse-writable computed fields, add `inverse='_set_field'` and `readonly=False`.

Additional computed-field parameters worth knowing:

- `precompute=True`: compute before record insertion (only when no explicit value or default is provided). Useful for one2many line fields created in batch.
- `compute_sudo=True`: recompute as superuser to bypass access rights (default True for stored fields).
- `recursive=True`: declare explicitly for recursive dependencies like `parent_id.X`.
- `search='_search_field'`: implement search on a non-stored or virtual field.

## Constraints

- SQL constraints are usually more efficient than Python constraints; when performance matters, prefer SQL.
- Tutorial-confirmed syntax (20.0): `_check_percentage = models.Constraint('CHECK(percentage >= 0 AND percentage <= 100)', 'The percentage ... should be between 0 and 100.')` — unique names use `UNIQUE(name)`.
- Upgrading can fail when existing rows violate the new constraint: `odoo.schema: unable to add constraint ... as CHECK(...)` — clean the offending data, then re-run `-u`.
- `@api.constrains('field', ...)` runs on a recordset whenever the listed fields change; raise `ValidationError` when violated and return nothing when all records pass.

- Use `@api.constrains` for cross-field or cross-record validation.
- Use `models.Constraint` (Odoo 20) for database-level constraints; `models.Index` for composite or custom indexes; `models.UniqueIndex` for unique indexes. Legacy `_sql_constraints` is the pre-20 syntax still seen in older codebases.
- Constraint/index attribute names must begin with `_` to avoid clashing with field names.

```python
@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for rec in self:
        if rec.date_start and rec.date_end and rec.date_start > rec.date_end:
            raise ValidationError(_("End date must be after start date."))

_unique_ref = models.Constraint("UNIQUE(ref, company_id)", "Reference must be unique per company.")
_name_idx = models.Index("(last_name, first_name)")
```

Constraint error messages can be plain strings (translated automatically) or functions taking `(env, diag)`.

## onchange and ondelete

`@api.onchange` runs only in the UI. It can warn the user and set field values but cannot raise blocking errors (use `return {'warning': {...}}` for non-blocking alerts). Dotted names (e.g., `partner_id.tz`) are not supported. Calling CRUD methods (`create`, `read`, `write`, `unlink`) on the onchange pseudo-recordset is undefined behaviour.

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.delivery_address_id = self.partner_id.address_get(['delivery'])['delivery']
```

`@api.ondelete` prevents deletion under business conditions. It survives module uninstallation (unlike overriding `unlink` directly).

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_user_inactive(self):
    if any(user.active for user in self):
        raise UserError(_("Can't delete an active user!"))
```

Set `at_uninstall=True` only when the check should apply during module uninstallation (rare; e.g., preventing deletion of the last language).

## Field-Level Access

Set `groups='module.group_name'` on a field to restrict access. Restricted fields are removed from views, `fields_get()`, and direct read/write attempts raise an AccessError.

## Domain Safety

Never concatenate raw domain lists from user input. Use the `Domain` object to combine domains safely.

```python
from odoo.osv import Domain  # verified: Domain class exists; see ORM reference domains section

# Bad: user can inject ['|', ('id', '>', 0)]
domain = user_provided + [('user_id', '=', self.env.uid)]

# Good
domain = Domain(user_provided)
domain &= Domain('user_id', '=', self.env.uid)
recs = self.search(domain)
```

## Index Types

The `index` field parameter accepts:

- `"btree"` or `True`: standard BTREE index (good for many2one).
- `"btree_not_null"`: BTREE without NULL values (useful when most values are NULL).
- `"trigram"`: GIN with trigrams (good for full-text search on Char/Text).
- `None` or `False`: no index (default).

## CRUD Overrides

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals['ref'] = self.env['ir.sequence'].next_by_code('my.model') or '/'
    records = super().create(vals_list)
    return records

def write(self, vals):
    if 'state' in vals and vals['state'] == 'done':
        vals['done_date'] = fields.Datetime.now()
    return super().write(vals)
```

Prefer `@api.model_create_multi` over `@api.model` for `create` to support batch creation.

## Mixins

- Mixins add behaviour through `_inherit` while keeping or adding a `_name`; combine several freely: `_inherit = ['mail.thread', 'mail.activity.mixin']`.
- `mail.thread`: add `<chatter/>` to the form view (options: `open_attachments`, `reload_on_attachment`/`reload_on_follower`/`reload_on_post`). Post from code with `message_post()` — the body is escaped when given as `str`, pass `markupsafe.Markup` for HTML. `tracking=True` on fields logs every change to the chatter.
- Conditional notifications: define `mail.message.subtype` records and override `_track_log_get_default_subtype(init_values)` to pick one based on what changed. `_notify_get_groups` customizes access/follow/action buttons per recipient group.
- `mail.activity.mixin` is NOT bundled with `mail.thread`: it adds schedulable activities (`mail.activity`), shown above the chatter and via the `kanban_activity` widget on `activity_ids`.
- `utm.mixin` adds `campaign_id`/`source_id`/`medium_id`, auto-filled from URL parameters via cookies; extend with `tracking_fields()`.
- `website.published.mixin` adds `website_published` + computed `website_url` (you must implement `_compute_website_url`); toggle it with an `oe_stat_button` + `website_button` widget. `website.seo.metadata` adds frontend metadata.
- Reference: `reference/backend/mixins` — also covers mail aliases (`mail.alias.mixin`) and customer ratings.

## Model Inheritance

- `_inherit = "other.model"` extends a model from another module: add fields, override field definitions, add constraints, add or override methods. It is the most-used inheritance mechanism.
- Delegation (`_inherits`) links each record to a parent record and gives transparent access to the parent fields; it is much rarer.
- Convention: each inherited model lives in its own Python file (`models/inherited_model.py`).
- When overriding methods: **always call `super()`** and return data **consistent with the parent** (dict returns dict). Use `super(Class, self)` only to call the parent with a modified recordset.

## Write Commands (One2many / Many2many)

- One2many/Many2many fields accept a **list of commands**, executed sequentially. Prefer the readable `Command` namespace over raw `(command, id, values)` triplets:

```python
from odoo import Command

self.env["account.move"].create({
    "partner_id": self.buyer_id.id,   # create() does not accept recordsets — pass ids
    "invoice_line_ids": [
        Command.create({"name": "Line", "quantity": 1.0, "price_unit": 100.0}),
    ],
})
```

- Helpers mirror the classic triplets 0-6: `create`, `update`, `delete`, `unlink`, `link`, `clear`, `set` — see the `odoo.Command` autodoc in the ORM reference.
- Link child lines at parent creation when possible — it is the simplest and most efficient way.

## Recordset Operations

```python
recs = self.env['my.model'].search([('state', '=', 'draft')])
recs = recs.filtered(lambda r: r.amount_total > 0)
names = recs.mapped('partner_id.name')
by_state = recs.grouped('state')  # Odoo 17+
```

- `.mapped('partner_id.name')` returns a list; `.mapped('partner_id')` returns a recordset.
- `.exists()` filters out stale records before further operations.
- Use `.browse(ids)` only when records are known to exist or in a controlled context.

## Relational Fields

- Naming conventions: `Many2one` fields end with `_id`; `One2many` and `Many2many` fields end with `_ids`.
- `One2many` is a **virtual** relationship (no database column) and requires a `Many2one` on the comodel pointing back: `fields.One2many(comodel, inverse_field)`. When records are created through a one2many field, the inverse many2one is populated automatically.
- `Many2many` is bidirectional many-to-many; the relation table is created automatically.
- Relational fields hold recordsets — ordered collections supporting `len()`, iteration, and set operations like `recs1 | recs2`.
- Child models accessed only through a parent (e.g. order lines) usually need no action or menu of their own.

## Workflow: State vs Stage

- Fixed linear process shared by all records: a `state` Selection + header buttons + `widget="statusbar"`; guard transitions with `UserError`/constraints. Renaming selection keys does not migrate existing rows — plan a data migration.
- User-configurable pipeline: a stage model (`_order = "sequence, id"`, `name`, `sequence`, `fold`) + `stage_id = fields.Many2one(..., group_expand="_read_group_stage_ids", index=True)`. Reference implementation: `crm.lead`.
- `group_expand` makes empty kanban columns visible: the named method re-fetches all stages (`stages.search([], order=order)`).
- Kanban grouped by a Many2one: dragging a card to another column writes the Many2one automatically.
- Give stages a technical key field (e.g. `stage_type` Selection) — business logic must not key off the translatable display name.
- Enforce stage transitions in a `write()` override (compare `sequence`), set milestone dates there, and add `tracking=True` on `stage_id` (with the `mail.thread` mixin) so the chatter logs every stage change.
- The ORM reference documents `_fold_name` (which field folds kanban columns) and `_read_group`.

## The Environment (`self.env`)

- `self.env.cr` / `self._cr`: database cursor
- `self.env.uid` / `self._uid`: current user id
- `self.env.user`: current user record
- `self.env.context` / `self._context`: context dictionary
- `self.env.ref(xml_id)`: record by external id
- `self.env[model_name]`: model instance

## Reserved Fields and Defaults

- `active`: archiving flag. Records with `active=False` are filtered out of searches by default. Gotcha: adding `active` to an existing model assigns `False` to all existing records — set `default=True` in the field definition.
- `state`: conventional `Selection` status field (usually `required=True`, `copy=False`, defaulting to the first state).
- `default=X` accepts a literal or a callable; pass the function itself without parentheses (e.g. `default=fields.Datetime.now`) or use a lambda. Date helpers: `fields.Date.today()`, `fields.Date.context_today(self)`, `fields.Date.add(date, months=3)` (helper signatures: see the `odoo.fields.Date` API in your installation).

## Ordering and Related Fields

- Define `_order = "id desc"` on the model; **without it, record order is non-deterministic (PostgreSQL-dependent)**. `_order` becomes the SQL `ORDER BY` and applies everywhere, including programmatic reads.
- Manual user ordering: a `sequence` Integer field + `handle` widget in the list view; the sequence field **must be first in `_order`**.
- `related="a.b.c"` is a shortcut equivalent to a compute with `@api.depends` on the path. Add `store=True` when it must be searchable/sortable or filtered by actions.

## Float Comparisons

Never compare floats with `==`, `<` or `>` directly — always use `odoo.tools.float_utils`:

```python
from odoo.tools import float_compare, float_is_zero
float_compare(a, b, precision_digits=2)  # -1 / 0 / +1
float_is_zero(x, precision_digits=2)     # bool
```

## Computed vs Onchange

- **Prefer computed fields** over onchange: computes also trigger outside form views (programmatic creates, imports, RPC).
- **Never put business logic in an onchange** — onchanges only fire in form views, not on programmatic record creation.
- Computed fields are read-only by default; add an `inverse` method to make them writable (inverse sets the dependencies and runs at save; compute runs on every dependency change).
- Computed fields are not stored, hence not searchable, unless `store=True` or a `search=` method is defined. `store=True` recomputes every referencing record when a dependency changes — evaluate the cost.
- `@api.depends` supports paths through relational fields (`partner_id.name`) and works for Many2many/One2many too.
- Onchange methods run on a single record in a form view (no `for record in self` loop) and may return non-blocking warning messages.
- Debugging: computed fields are easy to trace (one method sets the value); onchanges are hard — several may set the same field. Do not over-automate: end users get confused.

## Avoiding Common Pitfalls

- Never rely on mutable module-level (global) variables: one Odoo process can serve several databases with different modules installed.

- Never rely on mutable module-level (global) variables: one Odoo process can serve several databases with different modules installed.
- Do not iterate over a recordset and call `search` per record; batch or use `mapped`.
- `self.env.cr.execute()` bypasses the ORM and security; use it only for performance-critical reads and always parameterise.
- `_rec_name` controls the display name field; otherwise Odoo falls back to `name`.
- Set `tracking=True` on fields that need chatter logging (requires `mail` dependency).
- Use `copy=False` on fields that should not duplicate when duplicating a record.

## Automatic Fields

Every model automatically gets system-managed fields (readable, not writable): `id` (primary key), `create_date`, `create_uid`, `write_date`, `write_uid`. The automatic creation of log fields can be disabled per model via the log-access option (`_log_access`).

### read_group in 20.0: new signature, tuple rows (verified on source + live bug)

- 20.0 removed the old `read_group(domain, fields, groupby)` dict API. New: `read_group(domain, groupby=(), aggregates=['field:sum'])` returning `list[tuple]` (one tuple per group, aligned with groupby then aggregates; empty list when no rows).
- Un-tuple rows: `rows[0][0]` for a single aggregate. Never index with string keys.
- Tests can miss compute-field bugs: compute only runs when the field is READ (e.g. list render). Add assertions that READ computed fields, not just call actions.

### Two-tier stock design + flush-time UNIQUE (battle-tested)

- Two-tier inventory: `quant` (product, location) -> quantity for bulk goods; one record per physical piece (`unit`: serial barcode, state, location) for serialized goods. Pick ONE source of truth per product via a `tracking` selection: serialized products never write quants; on-hand = count of units.
- `models.Constraint` UNIQUE fires at FLUSH, not at create: code that continues after create won't see it (UI requests see it at flush/rollback). For business-inline uniqueness, pre-check with search_count + UserError, keep the DB constraint for race safety.
- Non-stored computed fields cache per transaction: after changing underlying data in the same transaction, call `invalidate_recordset()` before re-reading. UI requests are immune (fresh cache per request) — tests are not.

### Outbound state machines: when does stock actually leave?

- Map document states to unit/stock effects explicitly: allocation at confirm (units keep state in_stock but get a delivery_line_id), packed/shipped/received transitions follow the doc, and STOCK LEAVES AT SHIP (move with empty dest), not at done — done is the customer-received acknowledgment.
- Serial allocation: FIFO search on (product, state=in_stock, delivery_line_id=False) with limit=qty; raise when short. Cancel must release allocations (state back to in_stock, delivery_line_id=False).

### Models with optional location sides: company must coalesce (review-verified bug)

- A move-like model with optional source AND destination must NOT bind company via `related` to one side: `related='dest_location_id.company_id'` yields NULL for outbound rows, and a company-bound access rule then silently hides them. Compute company as `source.company_id or dest.company_id`, stored.
- Changing a `tracking`-like discriminator after transactional data exists forks the two sources of truth: guard with an `@api.constrains` that checks for existing rows (quants/units) before allowing the switch.
- 'Append-only ledger' needs an `@api.ondelete` guard, not just a cancel guard: CRUD ACLs otherwise allow deleting done rows.

### Scan-box pattern: hardware scanners without custom JS (verified)

- USB/Bluetooth scanners are keyboards: they type the code and press Enter. A non-stored Char field plus `@api.onchange` turns any form into a scan terminal — the decorator contract sends field assignments (clearing the box, writing feedback into another non-stored Char) back to the client automatically.
- Gate the next workflow button on scan completeness (raise UserError with remaining count); show progress via a compute like '2/3 scanned'. Scan-to-allocate: if the code matches an unallocated in-stock unit of a line's product, allocate on scan.
- Reset scan flags when the document is cancelled/released so the next wave starts clean.

### Operator trail on state machines + TransactionCase class-savepoint semantics

- Record WHO did each transition: a `*_by = fields.Many2one('res.users', readonly=True, copy=False)` per state, set from `self.env.user` inside the action method (with_user in tests to assert). Dates pair with them.
- 20.0 `TransactionCase`: ONE transaction per class, each test in a savepoint rolled back to the CLASS state (setUpClass effects persist as the baseline). Class fixtures that allocate/transition data collide with fresh-fixture tests — make such tests self-contained inside the method.

### Custom backend JS at bundle top: 'odoo.define is not a function' at boot

- Symptom: blank webclient, console `Uncaught TypeError: odoo.define is not a function` at the top of web.assets_web.min.js. Cause: a manifest `assets: {web.assets_backend: [...]}` entry got applied BEFORE web's module_loader.js, so the addon's transpiled `odoo.define(...)` registration ran before the shim assignment (verified in the live bundle: addon file at byte ~25, `odoo.define =` at ~6300).
- Fix: declare static assets through a data record instead - `<record model="ir.asset">` with `bundle=web.assets_backend`, `sequence=99` (ir.asset resolution: seq<16 first, then addon manifests, then seq>=16 last) - and keep the manifest free of an assets key.
- The bundle URL hash is content-independent (same hash across rebuilds), so deleting the cached `ir.attachment` bundles is required after reordering; the URL alone will not bust the cache.


### required=True on a NEW field + existing rows: no NOT NULL, no ORM check, constrains silent

- Adding a `required=True` Many2one during an upgrade to a table that already has rows: Odoo logs `WARNING odoo.schema: Missing not-null constraint on <table>.<column>` and creates the column **nullable** (can't set NOT NULL with NULLs present). The ORM performs NO python-level required check for relational fields on create — so RPC creates bypass it entirely.
- `@api.constrains("the_field")` does NOT save you: constrains fire only when the field appears in the create vals / is written. `create({"name": "x"})` without the field never triggers it.
- Real guard (verified v20.0.1.11.0): keep `required=True` (web form blocks empty save client-side) AND override create/write:
  ```python
  @api.model_create_multi
  def create(self, vals_list):
      for vals in vals_list:
          if not vals.get("the_field"):
              raise ValidationError("Message")
      return super().create(vals_list)
  def write(self, vals):
      if "the_field" in vals and not vals["the_field"]:
          raise ValidationError("Message")
      return super().write(vals)
  ```
- The NOT NULL appears on its own once the table is empty (fresh DBs / after data cleanup) — the create override is the portable, always-on layer. Test it by creating WITHOUT the key in vals, not by passing False.

### Category-level config cascading to records: related + store

- Pattern for "each record inherits a setting from its category" (e.g. every product's customs type comes from its material type): `Many2one(..., related="category_id.the_field", store=True, readonly=True, index=True)` on the record, and the actual `required` Many2one lives on the category. Stored related = searchable/groupable without joins; readonly prevents drift from the category source of truth. Changing the category's value updates stored related columns on recompute.
- One-code-per-record: `default=lambda self: self.env["ir.sequence"].next_by_code("wms.product")` with `copy=False` — sequence defined with `company_id=False` so codes are globally unique across companies.
