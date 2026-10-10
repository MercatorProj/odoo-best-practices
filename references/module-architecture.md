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
- Use `demo` only for sample records loaded with `--load-demo-data`.
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
