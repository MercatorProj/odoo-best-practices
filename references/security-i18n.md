# Security and i18n

## Access Rights

Every model needs an entry in `ir.model.access.csv`. Four permission levels: `perm_read`, `perm_write`, `perm_create`, `perm_unlink`.

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_model_user,my.model.user,model_my_model,base.group_user,1,1,1,0
access_my_model_manager,my.model.manager,model_my_model,my_group_manager,1,1,1,1
```

- Always grant at least read access to `base.group_user` unless the model is internal-only.
- Never leave a model without any access rules; it becomes invisible to all non-admin users.
- Use `perm_unlink: 0` for transactional data users should not delete.
- The UI hides the Create button when the user only has read permission.

## Multi-Company Architecture

- Single company = one shared data pool. Multi-company = transactional models carry a `company_id` and global rules isolate data by `company_ids` (the user's current companies; a user may be logged into several at once since 13.0).
- Per-company values on shared records: `company_dependent=True` fields; computes touching them need `@api.depends_context('company')`; read with `with_company(company)` — use it liberally in onchange/compute (`self = self.with_company(self.company_id)`).
- Cross-company consistency: `_check_company_auto = True` on the model + `check_company=True` on relational fields (never on `company_id` itself). Strict: a company-less record cannot link to a company-set one; without a domain, `['|', ('company_id', '=', False), ('company_id', '=', company_id)]` is applied implicitly.
- Make `company_id` required with `default=lambda self: self.env.company`; hide the field with `groups="base.group_multi_company"` (single-company users don't need it).
- Two canonical global rules: shareable `['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]`; restricted `[('company_id', 'in', company_ids)]`. Keep multi-company rules global — extra group rules would allow bypassing them.

## Translations (i18n)

- Ship an `i18n/` folder: `<module>.pot` template + `<lang>.po` files. Export the template from Settings → Translations → Import/Export → Export Translation (PO format). PO files load automatically when the language is installed and on every module install/update.
- Implicit exports: view text nodes plus `string`/`help`/`sum`/`confirm`/`placeholder` attributes; QWeb text nodes plus `title`/`alt`/`label`/`placeholder` (`t-translation="off"` to opt out); field `string`/`help`/selection values, and the stored values of `translate=True` fields.
- Explicit (code): Python `self.env._("literal")` (or legacy `odoo.tools.translate._`), JavaScript `_t`. Only **literal strings** are exportable — pass variables as lookup parameters (`_("Meeting with %s", name)`), never pre-format.
- Keep each full sentence in ONE lookup so translators can reorder; branch plurals as separate complete messages instead of appending letters.
- Never translate at import time (module-level dicts) — use `LazyTranslate(__name__)` / JS `_lt`, or look up inside the method.
- Module translations are backend-only by default: expose them to the website by prefixing the module with `website` or overriding `ir.http._get_translation_frontend_modules_name`.
- Stray spaces inside template text nodes break translation matching (see PDF Reports).

## Record Rules

Record rules filter which records a user can see, independent of model access.

```xml
<record id="my_model_company_rule" model="ir.rule">
    <field name="name">My Model: multi-company</field>
    <field name="model_id" ref="model_my_model"/>
    <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
</record>
```

- `domain_force` is evaluated with the current user's environment.
- Rules are OR-ed together for groups the user belongs to.
- Global rules (no `groups` field) are AND-ed with group rules.
- Global and group rulesets intersect: the first group rule added to a global ruleset restricts access further.

- Default-allow: if no rule applies to a model and operation, the operation is allowed — pair rules with correctly scoped ACLs.
- `perm_read`/`perm_write`/`perm_create`/`perm_unlink` scope a rule per operation (e.g. `perm_read="False"` makes the rule guard only create/write/unlink).
- Rules are complex, not bulk: write them in XML (not the ACL CSV); link groups with `eval="[Command.link(ref('base.group_user'))]"`.
- Multi-company rules are usually **global** (danger: a non-global one invites bypass through additional group rules); `company_ids` = all the user's companies, `company_id` = the active one.
- Security is checked only when data is accessed via the ORM. For non-CRUD operations or legitimate bypasses, check explicitly: `self.env.user.has_group(...)`, or `check_access(operation)` on the recordset (empty recordset checks general model access).
- Bypasses: `sudo()` ignores ACLs and rules (hard-coded group checks may still apply); raw SQL bypasses security as a side-effect. Use sparingly, validate input, comment why.

## Security Groups

```xml
<record id="module_category_myapp" model="ir.module.category">
    <field name="name">My App</field>
    <field name="sequence">25</field>
</record>

<record id="group_user" model="res.groups">
    <field name="name">User</field>
    <field name="category_id" ref="module_category_myapp"/>
    <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
</record>

<record id="group_manager" model="res.groups">
    <field name="name">Manager</field>
    <field name="category_id" ref="module_category_myapp"/>
    <field name="implied_ids" eval="[(4, ref('group_user'))]"/>
    <field name="users" eval="[(4, ref('base.user_root')), (4, ref('base.user_admin'))]"/>
</record>
```

## Multi-Company

- Add `company_id` field to company-specific models.
- Set `check_company=True` on `company_id` to auto-validate related records.
- Use `_check_company_auto = True` on the model for automatic checks on related fields.
- Record rules should always include the multi-company domain shown above.

## Public vs Private Methods

Any public method (no leading underscore) can be called via RPC with arbitrary parameters. ACLs are only verified during CRUD, not before method execution.

```python
def action_done(self):
    if self.state == 'draft' and self.env.user.has_group('base.manager'):
        self._set_state('done')

def _set_state(self, new_state):
    self.sudo().write({'state': new_state})
```

- Prefix internal methods with `_` to make them non-RPC-callable.
- Use `@api.private` to mark existing public methods as non-RPC.
- Never trust the record or arguments in a public method; validate before use.

## SQL Injection

- Never use string concatenation (`+`) or `%` formatting for SQL values.
- Use `self.env.execute_query(SQL(...))` or `cr.execute()` with parameterised tuples.
- Prefer `SQL` wrapper from `odoo.tools.sql` for readable, safe query construction.

## XSS Prevention

- Never use `t-raw` in QWeb templates; use `t-out` or `t-esc` instead.
- Use `Markup` from `markupsafe` to build safe HTML content.
- Sanitise `fields.Html` content with `sanitize=True` (default).

## Translations

- Wrap user-facing strings with `_()`: `raise UserError(_("Record not found."))`.
- In XML views, strings in `string`, `help`, and `sum` attributes are automatically extracted.
- Export `.po` files with `--i18n-export` and place them in the `i18n/` directory.
- File naming: `<lang>.po` (e.g., `zh_CN.po`).
- Do not hard-code language-specific formatting; use `lang` in context and Odoo's formatting helpers.

## sudo() and Elevated Access

- Use `sudo()` only when business logic requires bypassing access rules (e.g., system-level operations).
- Always add a comment explaining why `sudo()` is needed.
- Never expose `sudo()` records directly to user-facing actions without re-checking permissions.
- Prefer `with_user(user)` for testing permissions of a specific user.


## Public Methods and RPC Exposure

- Methods called by `type="object"` buttons are public (no `_` prefix) and callable through RPC — treat them as an attack surface.
- Always assume a public method runs on multiple records: loop `for record in self`.
- A public method must return a value so it stays XML-RPC-callable; `return True` when nothing better applies.
- Raise `UserError` (from `odoo.exceptions`) for user-facing validation instead of silently returning.

## View Hardening Is Not Security

- View-level `readonly` / `invisible` / `required` change only the user experience — no server-side check exists.
- Real enforcement: ACLs, record rules, field-level `readonly=True`, or constraints.

### Odoo 20 Access-Control Overhaul (verified on source)

- `ir.rule` and `ir.model.access` are GONE in 20.0, replaced by one model: `ir.access`. Fields: `name`, `model_id`, `group_id`, `operation` (selection like `crud`, `cru`, `r`, `cud`), `domain` (char).
- Semantics: `group_id` set = **permission** (adds access for the group); `group_id` empty = **restriction** (global limit for everyone).
- ACL files: `security/ir.access.csv` with header `id,name,model_id,group_id/id,operation,domain` (verified against the git blob of base).
- `res.groups`: `category_id` removed — groups hang off `privilege_id` (`res.groups.privilege`: name + `category_id` on `ir.module.category` + sequence). The `users` field is removed: membership is written from the user side (`res.users.group_ids` with `Command.link`).
- Rule domains still evaluate with `company_ids` (= `env.companies.ids`) via `ir.access._eval_context`.

### Multi-Company Rule Templates: shared vs company-bound (trap verified)

- Company-BOUND records (required company_id, e.g. documents): restriction domain `[('company_id', 'in', company_ids)]`.
- SHARED records (optional company_id, empty = global, e.g. products/categories): MUST use `['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]` — the bound template hides every empty-company row (`False in [1]` is False).
- Picking the wrong template silently hides data with zero errors — the UI just shows an empty list. Check the rule template first when a list is mysteriously empty.

### Odoo 20.0 module translations: i18n/<lang>.po is the only ship format

- In 20.0, `get_po_paths()` scans ONLY `i18n/<lang>.po` and `i18n_extra/<lang>.po`; there is no per-language CSV and no export wizard (jsonb era). CSV files under i18n are read only as data files (CSVDataFileReader).
- A po entry needs the right `#:` reference to land: `model:ir.model.fields,field_description:<module>.field_<model_underscored>__<field>`, `model:ir.model.fields.selection,name:<module>.selection__<model>__<field>__<value>`, `model_terms:ir.ui.view,arch_db:<module>.<view_xmlid>`, `model:res.groups,name:<module>.<xmlid>`.
- Python/JS terms additionally REQUIRE the marker comment `#. odoo-python` / `#. odoo-javascript` - CodeTranslations filters rows by that marker, without it every code term is silently dropped (symptom: field labels translate, env._() messages stay English).
- CodeTranslations is a per-process lazy cache read straight from the po file; after editing the po, RESTART (or re-upgrade) - a long-running worker keeps its cached dict.
- Raw strings bypass env._(): sweep user-facing paths (onchange feedback, RPC return dicts) for `= "..." % x` patterns and wrap them, or they never translate.
- res.lang rows often pre-exist inactive; search with with_context(active_test=False) and set active=True (20.0 res.lang has no decimal_separator/thousands_sep fields).
- Field labels live in ir_model_fields.field_description jsonb ({'en_US': ...}); read them with raw psycopg2 as ground truth when reconstructing definitions - and NEVER regex-sweep `string=` into field defs: Many2one/One2many first positional args are comodel/inverse, not labels. Parse top-level chunks with a string-aware scanner and verify with py_compile (doraise) run with the file list passed through an env var (cmd mangles non-ASCII path arguments).
- Selection labels ship as `model:ir.model.fields.selection,name:<module>.selection__<model_underscored>__<field>__<value>` entries; sweep the WHOLE bracketed `selection=[...]` region (defs are usually multi-line) and note `_name` lines may be indented. If a wrong value was already imported, jsonb keeps the zh key and plain -u will NOT fix it - re-run with `--i18n-overwrite`. To render a translated state label server-side use `field._description_selection(env)` (lives in fields_selection.py).
