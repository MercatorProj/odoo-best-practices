# Testing and Performance

## Test Structure

```python
from odoo.tests import TransactionCase, tagged
from odoo.exceptions import ValidationError


@tagged('post_install', '-at_install')
class TestMyModel(TransactionCase):

    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        cls.partner = cls.env['res.partner'].create({'name': 'Test Partner'})
        cls.model = cls.env['my.model']

    def test_create_basic(self):
        rec = self.model.create({
            'name': 'Test',
            'partner_id': self.partner.id,
        })
        self.assertEqual(rec.state, 'draft')
        self.assertEqual(rec.partner_id, self.partner)

    def test_constraint_rejects_invalid(self):
        with self.assertRaises(ValidationError):
            self.model.create({'name': ''})
```

- `TransactionCase`: each test runs in a rolled-back transaction (most common).
- `SavepointCase`: all tests share a single transaction with savepoints (faster for setup-heavy tests).
- `HttpCase`: for testing controllers and tour-based frontend tests.
- Use `@tagged('post_install', '-at_install')` for tests that need the module fully installed.
- Test files go in `tests/` with an `__init__.py` that imports them.

## Frontend Tour Tests

```python
from odoo.tests import HttpCase, tagged


@tagged('post_install', '-at_install')
class TestUi(HttpCase):

    def test_01_ui(self):
        self.start_tour("/web", 'my_model_tour', login="admin")
```

Tours are defined in JavaScript and registered in the `web_tour.tours` registry.

## Performance Patterns

### Batch Operations

```python
# Good: single write on the recordset
recs.write({'state': 'done'})

# Bad: per-record write in a loop
for rec in recs:
    rec.write({'state': 'done'})
```

### read_group for Aggregates

```python
groups = self.env['my.model']._read_group(
    domain=[('state', '=', 'done')],
    groupby=['partner_id'],
    aggregates=['amount_total:sum'],
)
# Returns [(partner_record, sum_value), ...]
```

### SQL Indexes

- Add `index=True` to fields used in frequent search domains (`name`, `partner_id`, `state`).
- For composite indexes, use `init()`:

```python
def init(self):
    self.env.cr.execute("""
        CREATE INDEX IF NOT EXISTS my_model_state_company_idx
        ON my_model (state, company_id)
    """)
```

### Prefetch

The ORM automatically prefetches fields on recordset iteration. Calling `.mapped()` or `.filtered()` benefits from this. Avoid `rec.field` inside a loop when `recs.mapped('field')` gives the same result in one query.

### Avoiding N+1

```python
# Good
partners = recs.mapped('partner_id')
for partner in partners:
    print(partner.name)  # Single batched read

# Bad
for rec in recs:
    print(rec.partner_id.name)  # May trigger separate queries
```

## Running Tests

```bash
odoo-bin -d test_db -i my_module --test-enable --stop-after-init
odoo-bin -d test_db -u my_module --test-enable --test-tags /my_module --stop-after-init
```

- `--test-tags /my_module` runs only tests from that module.
- `--test-tags /my_module:TestMyModel` runs a specific test class.
- Tests run in a separate transaction that is rolled back; no data persists after the run.

## Profiling

Odoo includes a built-in profiler for identifying performance bottlenecks. It records SQL queries and stack traces, viewable as flamegraphs in the built-in speedscope viewer.

Enable globally in developer mode (Settings > Performance), or profile specific code with:

```python
from odoo.tools.profiler import Profiler

with Profiler(collectors=['sql', 'traces_async']):
    do_stuff()
```

Three collectors: `sql` (SQL queries with stack), `traces_async` (periodic stack sampling), `qweb` (template directive timing). Combine with `assertQueryCount` in tests to catch regressions.

Profiling tips:

- Disable the profiler when measuring real performance impact; it adds overhead.
- Use `ExecutionContext` to break down results of repeated calls.
- QWeb collector is useful for optimising view rendering.
