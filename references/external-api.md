# External API (JSON-2, 19.0+)

## Endpoint and Authentication

- Legacy `/xmlrpc`, `/xmlrpc/2` and `/jsonrpc` are deprecated since 19.0: the `db` service was removed in Odoo 20, `common`/`object` are scheduled for removal in Odoo 22. `@route(type='jsonrpc')` controllers are NOT affected. New integrations use the JSON-2 API.
- Call any public model method: `POST /json/2/<model>/<method>` with header `Authorization: bearer <API KEY>`; add `X-Odoo-Database` only when the domain hosts several databases; set `Content-Type: application/json` and a descriptive `User-Agent`.
- Body is a JSON object of **named arguments only** (no positional), plus `ids` (recordset methods) and `context`. `@api.model` methods omit `ids`.
- Errors return 4xx/5xx with `{name, message, arguments, context, debug}`.
- `GET /web/version` replaces the old `version()` call; current user via `res.users/context_get`.

## API Keys and Users

- Keys live at most 3 months — rotate; generate programmatically with `res.users.apikeys.generate` (needs `base.enable_programmatic_api_keys`; 10-key default limit → HTTP 422) and revoke with `revoke`. Revoke the old key only after switching services to the new one.
- For long-lived integrations use dedicated bot users with minimal rights (and no interactive password); all operations are still validated against ACLs, record rules and field access of that user.
- The external API requires a Custom pricing plan on Odoo Online; self-hosted Community is unrestricted.

## Transactions

- Each JSON-2 call runs in its own SQL transaction: committed on success, rolled back on error. Multiple calls cannot share a transaction — never build cross-call invariants (payments, reservations).
- Put multi-step business logic in a single model method (often `action_*`) and call that; prefer `search_read` over `search` + `read`.

## Integration Topology

- Inbound (others call Odoo): no separate server is built — Odoo already IS an HTTP server. Use the built-in `/json/2` endpoints with an API key, or add custom routes inside a module with `@route` controllers (same process, same transaction semantics).
- Outbound (Odoo calls others): plain HTTP client code inside your module (`requests` + `ir.cron` for polling, `ir.config_parameter` for third-party secrets, `base_automation` for event-driven webhooks).
- A standalone middleware service (BFF/gateway) is an opt-in architecture for protocol translation or aggregating several backends — not required for typical integrations.

## Controllers (Custom Inbound Routes)

- Skeleton: `class MyController(http.Controller):` + methods decorated with `@route('/path', auth=...)`. Controllers have no `_name`; extensions subclass the controller class and MUST re-decorate overrides with `@route()` (no args keeps previous options) — an undecorated override silently unpublishes the route.
- Route options verified in 20.0 source (`odoo/http/routing_map.py`): `type` `'http'` or `'jsonrpc'` (`'json'` is a deprecated alias since 19.0); `auth` `'user'` | `'bearer'` | `'public'` | `'none'`; `bearer_scope` (required for bearer, restricts which token scopes may call); `methods` (verbs); `cors`; `csrf` (default on for http, off for jsonrpc); `readonly` (serve from a read-only replica cursor); `captcha`; `save_session`; `handle_params_access_error`.
- Security semantics: `auth='public'` runs as the shared Public user — ACLs and record rules still apply; `'none'` only for framework/auth bootstrap. Request context lives on `odoo.http.request`.
- Odoo 20 note: the HTTP layer is now the `odoo/http/` package (dispatcher, requestlib, response, router, server, session).
