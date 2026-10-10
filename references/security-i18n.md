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
