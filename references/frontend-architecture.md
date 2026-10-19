# Web Client Frontend Architecture

## Terminology and Code Map

- "component" always means OWL components; "widget" means legacy pre-Odoo-15 widgets. All new development should be done in Owl.
- The web client is an OWL single-page app served at `/web`; the `web/` addon sources live in `web/static/src` and import with the `@web` prefix: `core/` (low level), `fields/` (field components), `views/` (JS view components), `search/` (control panel, search bar, search panel), `webclient/` (navbar, user menu, action service).
- WebClient template: `<NavBar/> + <ActionContainer/> + <MainComponentsContainer/>`. The action service keeps a stack of active actions (the breadcrumbs); `MainComponentsContainer` renders everything registered in the `main_components` registry — the standard web-client extension point.
- Shared env keys: `qweb`, `bus`, `services`, `debug`, `_t`, `isSmall`.

## Contexts

- User context (from the `user` service): `allowed_company_ids` (multi-company switcher), `lang`, `tz`.
- Action context: `search_default_<filter>` toggles saved filters, `additional_context` is passed through `doAction` — it is a security risk if it can reach the server: `allowed_company_ids` must come from the user service, never from the action context.

## py_js: Browser-side Python Expression Evaluator

- View modifiers (`invisible`, `readonly`, `required`) and attrs are Python expressions evaluated in the browser by `@web/core/py_js`: `tokenize` → `parse` → `evaluate(ast, context)` or `evaluateExpr(expr, context)`.

## The Domain Class (@web/core/domain)

- Domains are lists, string expressions, or `Domain` instances; string expressions may embed Python (e.g. `context_today()`).
- `new Domain(descr).contains(record)` evaluates membership; `.toList(context)` substitutes free variables; `.toString()` serializes.
- Combinators: `Domain.and([...])`, `Domain.or([...])`, `Domain.not(d)`, `Domain.combine([...], 'AND'|'OR')` — they accept mixed list/string/Domain descriptions.

## Frontend Decision Guide

- Terminology trap: Odoo docs call the public website the "frontend" and the web client the "backend" — the reverse of usual browser/server parlance.
- Use OWL components when a UI must react to input without page reloads; static content needs neither OWL nor QWeb.