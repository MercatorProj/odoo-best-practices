# Module Architecture

## Directory Structure

Use this layout for custom modules. Omit directories that are not needed.

```
mycompany_feature/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   └── model_name.py
├── views/
│   └── model_name_views.xml
├── security/
│   ├── ir.model.access.csv
│   └── security_rules.xml
├── data/
│   └── data.xml
├── wizard/
│   ├── __init__.py
│   └── wizard_name.py
├── controller/
│   ├── __init__.py
│   └── main.py
├── report/
│   └── report_templates.xml
├── static/
│   ├── description/
│   │   └── icon.png
│   └── src/
│       ├── js/
│       ├── css/
│       └── xml/
├── tests/
│   ├── __init__.py
│   └── test_feature.py
└── i18n/
    └── zh_CN.po
```

## Manifest

```python
{
    "name": "Feature Name",
    "version": "20.0.1.0.0",
    "category": "Category",
    "summary": "Short one-line description",
    "description": "Longer markdown description",
    "author": "Author Name",
    "website": "https://example.com",
    "license": "LGPL-3",
    "depends": ["base", "mail"],
    "data": [
        "security/security_groups.xml",
        "security/ir.model.access.csv",
        "security/security_rules.xml",
        "data/data.xml",
        "views/model_name_views.xml",
    ],
    "demo": [
        "data/demo_data.xml",
    ],
    "assets": {
        "web.assets_backend": [
            "mycompany_feature/static/src/js/**/*",
            "mycompany_feature/static/src/css/*.scss",
        ],
    },
    "installable": True,
    "application": False,
    "auto_install": False,
}
```

Key rules:

- `depends` must list all modules whose models or views are extended or referenced.
- Load security files (`ir.model.access.csv`) before data and view files in the `data` list.
- Use `demo` only for sample records loaded with `--with-demo`.
- Prefer CSV over XML for simple, bulk data: loading a CSV file is faster than loading an XML file (official data-files guidance).
- Set `application: True` only for modules that appear in the Apps menu as standalone apps.
- Use `auto_install: True` sparingly; it installs the module when all dependencies are present.
- `auto_install` can also be a list of dependency names; the module installs when all listed dependencies are present (remaining deps auto-install too).
- Use `external_dependencies` to declare Python modules or system binaries required by the module.
- `pre_init_hook`, `post_init_hook`, and `uninstall_hook` accept function names from `__init__.py` for setup/cleanup that cannot be done through the API.

## Python Init Chain

```python
# __init__.py (root)
from . import models
from . import wizard
```

```python
# models/__init__.py
from . import model_name
```

## Extension vs New Model

- New model: define `_name` without `_inherit`.
- Extend existing model: use `_inherit = 'existing.model'` without `_name`.
- Prototype (copy with new table): use both `_name = 'new.model'` and `_inherit = 'existing.model'`.
- Delegation inheritance: use `_inherits = {'other.model': 'related_field_id'}` for composition.

## Migration

For schema or data changes between versions, add migration scripts:

```
migrations/
└── 20.0.1.0.0/
    ├── pre-migrate.py
    └── post-migrate.py
```

- `pre-migrate.py`: rename columns, drop old constraints, prepare data before schema update.
- `post-migrate.py`: populate new fields, clean up, recompute.

Both receive `(cr, version)` (Odoo <18) or `(env, version)` (Odoo >=18).


## Data Files: Master vs Demo

- Manifest: `data` for master data (always installed — views, actions, business base data), `demo` for demo data (loaded only in demo databases, `--with-demo`). Conventions: `data/`, `demo/`, `security/`, `views/` folders; CSV and XML load in the same sequence.
- CSV suits long lists of simple records (link columns suffixed `:id`); XML is the default for anything else.
- XML data superpowers: re-declaring an existing `xml_id` extends it (populates new fields on records from dependency modules); `ref` links by xml_id; `eval` computes values; `search` resolves records via the ORM; `<function>` calls model methods during load; x2many values use `Command.create`/`link` inside `eval`.
- `noupdate="1"` records are created if missing but not rewritten on module upgrade (`-i` bypasses this).
- Python access: `env.ref(xml_id)`. Never use demo data outside its declaration — not even in tests. Users can delete any record: code defensively around referenced data.
- Raw SQL imports bypass the ORM, computed fields, constraints and ACLs — last resort for huge datasets only. xml_id resolution lives in `ir.model.data`.

## Link Modules

- To glue two independently installable features (e.g. real estate + invoicing), create a **link module** whose `depends` lists both (`["estate", "account"]`).
- The base modules stay independent; the link module only exists when both are installed. Uninstalling a dependency uninstalls the link module too.
- To learn how to create records of a foreign model (e.g. `account.move`), read how other modules do it — there is no shortcut past reading the model and its existing flows.

## Pre-Release Polish Checklist

- Before releasing a module, refactor against the official coding guidelines and run a linter:
- module structure, variable names, method naming convention (`_private` vs public), model attribute order, XML ids.
- Multi-box packing pattern (wms_core v1.8.0): a `wms.box` child model (sequence BOX/, open/sealed state, One2many units), units carry `box_id`; the scan RPC takes an optional box_id, a scanned box barcode switches the "current box" client-side; `action_pack` enforces "if boxes exist, every scanned unit must be boxed" then auto-seals open boxes. Register new test modules in tests/__init__.py - an unregistered test file silently never runs.
- QWeb reports without wkhtmltopdf: ship as `qweb-html` (browser print-to-PDF), switch report_type to `qweb-pdf` once the binary exists. Report text nodes are po terms via `model_terms:ir.ui.view,arch_db:<module>.<template_id>` - extract them from `<template>` blocks, not just record attrs. XML load order matters: files defining `ir.actions.report` must load BEFORE any view whose button uses the `%(xmlid)s` action reference, or the registry fails with an id_get error.
