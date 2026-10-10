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

- Use `@api.constrains` for cross-field or cross-record validation.
- Use `models.Constraint` (Odoo 20+) for database-level constraints; the legacy `_sql_constraints` list is still supported but deprecated.
- Use `models.Index` (Odoo 20+) for composite or custom database indexes.

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
from odoo.osv import Domain

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

## Avoiding Common Pitfalls

- Do not iterate over a recordset and call `search` per record; batch or use `mapped`.
- `self.env.cr.execute()` bypasses the ORM and security; use it only for performance-critical reads and always parameterise.
- `_rec_name` controls the display name field; otherwise Odoo falls back to `name`.
- Set `tracking=True` on fields that need chatter logging (requires `mail` dependency).
- Use `copy=False` on fields that should not duplicate when duplicating a record.
