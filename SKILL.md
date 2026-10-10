---
name: odoo-best-practices
description: Apply Odoo 18-20 best practices for module architecture, ORM patterns, XML views, OWL components, security, testing, and performance. Use when developing, reviewing, or debugging Odoo custom modules.
---

# Odoo Best Practices

Apply these practices when building, extending, or reviewing Odoo modules targeting versions 18-20. The guidance assumes Odoo 20.0 conventions unless the user specifies an older version.

## Core Rules

- Never modify core modules. Always extend via `_inherit` in a custom module.
- Every new model must define `_description` and have access rules in `ir.model.access.csv`.
- Use `api.onchange` for UI-only validation and `api.constrains` for database-level invariants.
- Prefix custom module names with a vendor or project namespace (e.g., `mycompany_feature`) to avoid collisions.
- Set `version` in `__manifest__.py` to match the Odoo series (e.g., `20.0.1.0.0`).
- Mark user-facing strings with `_()` for translation support.
- Use recordset methods (`mapped`, `filtered`, `sorted`) over Python loops when possible to leverage ORM prefetch.
- Prefer `sudo()` sparingly; always document why elevated access is needed.
- Never concatenate domains from user input; use the `Domain` object (`domain &= Domain(...)`) to prevent injection.
- Use `models.Constraint` / `models.Index` (Odoo 20+) instead of legacy `_sql_constraints`.

## Routing

Read the relevant reference when the task touches that area. Do not load all references by default.

- **Module setup, manifest, migration:** read [references/module-architecture.md](references/module-architecture.md).
- **Model definitions, computed fields, constraints, recordsets:** read [references/orm-patterns.md](references/orm-patterns.md).
- **XML views, view inheritance, OWL components, frontend assets:** read [references/views-owl.md](references/views-owl.md).
- **Access rules, record rules, multi-company, translations:** read [references/security-i18n.md](references/security-i18n.md).
- **Test framework, performance patterns, query optimization:** read [references/testing-performance.md](references/testing-performance.md).
