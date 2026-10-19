# Testing (Odoo 20.0 verified)

## Runner

- odoo-bin -d <db> -u <module> --test-enable --test-tags /<module> --stop-after-init; result line: odoo.tests.result: 0 failed, 0 error(s) of N tests. failed = assertion, error = crash (fix crashes first).
- Tags: /module, /module:TestClass, /module:TestClass.test_method.
- Tests live in <module>/tests/ + every file imported in tests/__init__.py (a missed import silently skips the whole file).

## Transaction semantics (20.0 source-verified)

- TransactionCase: ONE transaction per class; each test method runs in a savepoint rolled back TO THE CLASS STATE. setUpClass side effects (allocations, transitions) are the baseline for every test - tests needing fresh fixtures must create them inside the method.
- Non-stored computes need invalidate_recordset() before re-reads in tests (web client is immune - it re-fetches).
- Never commit in tests; with_user() for other-operator assertions.

## Design

- Layers: method logic / lifecycle walks / guards (UserError = business guard, ValidationError = data integrity - assertRaises must match) / isolation (multi-company visibility, operator trail).
- Regression test after every fixed bug; contract change updates affected tests in the same change (e.g. new scan precondition on action_pack).
- api.constrains fires only for fields present in vals - pass optional fields explicitly (even False) to trigger them in tests.
- UNIQUE fires at flush, not create: keep inline search_count pre-checks for user-facing errors and assert that.
- Deterministic: fixed creation order (FIFO), no exact timestamps; ASCII-only files on Windows.

## Official-suite practices (verified against odoo 20.0 source + official test suites)

- Default test tags are {standard, post_install}; the framework enforces at_install XOR post_install per test class. Choose at_install to run during your own module's -u; post_install when you depend on the full registry (menus, other modules, demo data).
- Tag 'standard' runs in normal CI; tag 'extended' marks heavy tests opt-in via @tagged('-standard', 'extended').
- odoo.tests.Form (from odoo.tests import Form) is the official way to test onchange flows server-side: it runs default_get + onchange cycles and x2many proxies like the real client. Do not call _onchange_* methods directly when the flow matters.
- with_user() multi-perspective tests are the heaviest official idiom; assertLogs replaces the removed mute_logger for asserting expected error logs (mute_logger does not exist in 20.0).
- HttpCase + start_tour for JS/E2E (workstations, kanban drag, scan terminals).
- setUpClass fixtures: factory helpers with defaults (mail_new_test_user pattern) instead of raw create blobs repeated per test.
- flush_model/flush_recordset and invalidate_recordset are used deliberately around constraint and compute assertions.
- Test-tags grammar (tag_selector): [+-] prefixes, wildcard *, /module, :Class, .method; '-' removes a tag.
- Backend tests cannot catch UI crashes: a green 56-test gate shipped a station that died on menu click. Keep a Playwright rig (dedicated venv, NOT the odoo venv) and verify every client-action change through the real path: login -> deep-link `/odoo/action-<module>.<xmlid>` -> assert rendered DOM + collect console/pageerror. One Playwright run finds what 56 ORM tests cannot.
