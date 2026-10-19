# odoo-best-practices

A [Codex skill](https://platform.openai.com/docs/guides/codex) that applies Odoo 18-20 best practices for module architecture, ORM patterns, XML views, OWL components, security, testing, and performance. Content is aligned with the [official Odoo 20.0 documentation](https://www.odoo.com/documentation/20.0/index.html).

## What It Covers

| Reference | Topics |
|---|---|
| `module-architecture.md` | Directory structure, `__manifest__.py`, init chain, `_inherit` vs `_name`, migration scripts, hooks |
| `orm-patterns.md` | Model definition, computed fields, `models.Constraint` / `models.Index`, `@api.onchange`, `@api.ondelete`, recordsets, `Domain` safety, index types |
| `views-owl.md` | XML views (form/list/search), view inheritance, OWL 2 components, actions and menus |
| `security-i18n.md` | ACLs, record rules, security groups, multi-company, public vs private methods, SQL injection, XSS, translations |
| `testing-performance.md` | `TransactionCase`, `HttpCase`, tours, batch operations, `_read_group`, profiling |
| `dev-environment.md` | Source install, `odoo-bin` options, `--dev` features, config file, scaffolding, running the server |
| `official-docs.md` | Curated official documentation index (setup, tutorials, reference, howtos, guidelines) |

## Install

With [npx skills](https://github.com/vercel-labs/skills):

```bash
npx skills add MercatorProj/odoo-best-practices
```

Or copy manually into your Codex skills directory:

```bash
git clone https://github.com/MercatorProj/odoo-best-practices.git "$CODEX_HOME/skills/odoo-best-practices"
```

## Usage

The skill activates automatically when Codex detects an Odoo development task (module creation, model/view/security work, debugging, code review). You can also invoke it explicitly with `$odoo-best-practices`.

`SKILL.md` loads core rules and routes to the relevant reference file based on the task area. You do not need to read all references for every task.

## Requirements

- Codex CLI or any agent that supports the Codex skill format
- Odoo 18, 19, or 20 development context (defaults to Odoo 20 conventions)

## License

[MIT](LICENSE)

## Design Philosophy

The rules in this skill codify Odoo's design philosophy:

1. **Everything is a module** — even Enterprise features are modules stacked on Community.
2. **Data is the program** — views, actions, menus, and security are database records installed from XML data files.
3. **Extend, never modify** — inherit models, views, and templates; keep the core pristine for painless upgrades.
4. **XML IDs are the glue** — `module.xml_id` is the global reference namespace; `depends` works like a system package manager.
5. **Convention over configuration** — names map automatically to tables, columns, and UI.
6. **The ORM is the API and the security boundary** — public methods are RPC-exposed; raw SQL bypasses security.
7. **Integrated by design** — one database, shared base models; OWL is the frontend direction.

## Official Documentation

- Odoo 20.0 developer docs: https://www.odoo.com/documentation/20.0/developer.html
- Setup guide: https://www.odoo.com/documentation/20.0/developer/tutorials/setup_guide.html
- CLI reference: https://www.odoo.com/documentation/20.0/developer/reference/cli.html
- Source install (Python 3.12+): https://www.odoo.com/documentation/20.0/administration/on_premise/source.html
- Coding guidelines (XML ID naming): https://github.com/odoo/documentation/blob/20.0/content/contributing/development/coding_guidelines.rst
- Full curated index: [references/official-docs.md](references/official-docs.md)
