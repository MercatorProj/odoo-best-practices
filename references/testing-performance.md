# Testing and Performance

## Runbot (Odoo CI)

- Odoo's CI server: every commit, branch and PR is tested against regressions; passing runs are deployed with demo data at runbot.odoo.com — useful for exploring any app against demo data.

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
- Subclasses of `odoo.tests.BaseCase` (typically via `TransactionCase`/`HttpCase`) are automatically tagged `standard` and `at_install`; adjust with `@tagged(...)` (e.g. `@tagged('post_install', '-at_install')`).
- `HttpCase`: for testing controllers and tour-based frontend tests.
- Use `@tagged('post_install', '-at_install')` for tests that need the module fully installed.
- Test files go in `tests/` with an `__init__.py` that imports them.

## Test Authoring Rules

- Layout: a `tests/` folder at the module root; files named `test_*.py`, imported **only** in `tests/__init__.py` (never in the module `__init__.py`).
- Default tags: subclasses of `odoo.tests.BaseCase` (e.g. `TransactionCase`, `HttpCase`) are tagged `standard` + `at_install` (testing reference). Opt into `@tagged('post_install', '-at_install')` when the behaviour may be altered by later module installs. (Note: the 20.0 unit-tests tutorial's comment claims `post_install` is the default — the testing reference says otherwise; trust the reference.)
- `setUpClass` builds shared data once (faster than `setUp` per test); split suites per scope and share a `Common` base class like `account/tests/common.py`.
- Rules: stay independent of demo data; the framework rolls back after each test so **never call `cr.commit()`** (in tests or business code); a bug-fix test must fail before the fix and pass after; don't test the ORM — test business flows; no manual flushing.
- `onchange` fires only in form views: emulate one with `odoo.tests.Form`.
- Assertions: `self.assertRecordValues(records, [{...}, ...])` for field-by-field checks; `with self.assertRaises(UserError):` for guards.
- CLI: `--test-tags` implies `--test-enable`; spec format `[-][tag][/module][:class][.method]`; run a single file with `--test-file=path`.

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

### 20.0 Performance Additions

- Per-record count computes: replace `search_count` in loops with one grouped query — `counts = dict(other_model._read_group([('related_id', 'in', self.ids)], ['related_id'], ['__count']))`, then `record.count = counts.get(record, 0)`.
- Batch `create` too: build a list of dicts and call `model.create(values_list)` once.
- `with_prefetch()` controls or disables batch prefetching when browsing ids one at a time is unavoidable.
- Complexity: index lookups with a dict instead of nested loops; membership tests on a `set` (a list makes `in` quadratic); recordset math (`self - invalid_ids`) can replace id loops.
- Speedscope views: `sql (no gap)` for pure SQL optimisation, `sql (density)` to spot zones of many small queries worth batching.
- Profiler pitfalls: blocking C calls delay GIL release and show as long frames; long sessions may need `--limit-memory-hard`.
- Index tradeoff: every index slows `INSERT`/`UPDATE`/`DELETE` — do not index everything.

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
