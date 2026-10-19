# Development Environment Setup

Verified against the official Odoo 20.0 [setup guide](https://www.odoo.com/documentation/20.0/developer/tutorials/setup_guide.html) and [CLI reference](https://www.odoo.com/documentation/20.0/developer/reference/cli.html).

## Source install (preferred for development)

The official setup guide recommends running Odoo from source for development. The data tier is PostgreSQL only.

1. Install PostgreSQL.
2. Clone the server source: `git clone https://github.com/odoo/odoo.git -b 20.0` (community). Enterprise sources need a subscription and are not required for custom modules.
3. Create a virtualenv with a supported Python version, then `pip install -r requirements.txt`. Odoo 20 requires **Python 3.12 or later** (minimum raised from 3.10 to 3.12 in version 20, per the official install guide).
4. Keep custom modules in their own addons directory outside `odoo/addons`.

Typical workspace layout (repo locations are not crucial; keep them side by side):

```text
odoo-dev/
├── odoo/             # odoo/odoo clone (branch 20.0)
├── custom-addons/    # your modules
└── odoo.conf
```

## Running the server

```console
$ python odoo/odoo-bin --addons-path="odoo/addons,custom-addons" -d dev-db
```

- Fresh database: log in at `http://localhost:8069` with `admin` / `admin`.
- `-d <db>`: target database (created automatically).
- `-i mod1,mod2` / `-u mod1,mod2`: install / update modules at startup (require `-d`).
- `--addons-path <dirs>`: comma-separated directories scanned for modules.
- `--limit-time-cpu` / `--limit-time-real`: raise while debugging so slow requests are not killed.
- `?debug=assets` additionally serves unminified JS/CSS assets for debugging the web client.
- `-c <file>`: config file (also via `ODOO_RC`); `-s` saves current options into it. Windows lookup includes `%LOCALAPPDATA%\odoo\odoo\odoo.conf` and a config next to `odoo-bin`.
- `--with-demo` / `--without-demo`: demo data for new databases (default: without).

## Developer mode and --dev features

- Enable developer mode in the UI (Settings > General Settings > Developer Tools, or `?debug=1`). Required for the `Update Apps List` button.
- `--dev <features>` (development only, never production). Odoo 20 features:
  - `all` = alias for `xml,reload,qweb,access`
  - `xml`: read QWeb templates from XML files instead of the database
  - `reload`: restart the server when Python files change
  - `qweb`: break in QWeb evaluation on `t-debug="debugger"` nodes
  - `werkzeug`: display full traceback on frontend error pages
  - `access`: log the traceback next to AccessError-induced 403 responses
  - `replica`: simulate a DB replica host against the same server (read-only testing)

## Creating a module

Two officially supported ways:

1. **Manual (used by the official Server Framework 101 tutorial)**: create `<addons>/<module>/` with at least `__manifest__.py` (required key `name`; usually `depends: ['base']`) and `__init__.py` (may be empty). Restart the server, then Apps > Update Apps List (developer mode required).
2. **Scaffold**: `odoo-bin scaffold <name> [destination] [-t <template_dir>]` bootstraps a skeleton module to edit. Scaffolding is a convenience, not a substitute for understanding the manifest and directory rules.

`--skip-auto-install` helps check whether modules depend indirectly on auto-install modules.

## Standard development loop

- Python file changes: restart the server (or use `--dev reload`).
- XML view changes: launch with `--dev xml` and a browser refresh is enough — no server restart needed while iterating on views.
- Model, view, data, or manifest changes: restart **and** upgrade with `-u <module>`; `-u` must always be combined with `-d`.
- New empty models log warnings at startup: `The model X has no _description` and `The model X has no access rules, consider adding one...` — both should be resolved (`_description` attribute; ACL file).

## Production Deployment

- Windows runs the multi-threaded server only; production sizing assumes Linux multi-processing, enabled with `--workers > 0`. Rule of thumb: `(CPU * 2) + 1` workers, 1 worker ≈ 6 concurrent users; RAM ≈ workers * ((0.8 * 150MB) + (0.2 * 1GB)). Tune `limit_memory_hard`/`limit_memory_soft`, `limit_time_cpu`, `limit_time_real`, `limit_request`.
- Multi-database deployments must set `db_filter` (e.g. `^%d$` per subdomain), then `list_db = False` (or `--no-database-list`) and block `/web/database/*` routes at the proxy.
- `admin_passwd` (database manager master password) can only be set in the config file — generate randomly, it is stored pbkdf2-hashed; reset via temp plaintext → restart → Set Master Password in `/web/database/manager`.
- PostgreSQL: run Odoo with a dedicated non-superuser `db_user`; keep databases owned by another user; set `db_sslmode` for remote connections.
- HTTPS is mandatory (auth travels in cleartext): nginx SSL termination + `proxy_mode = True`; route `/websocket/` to the gevent port (8072); add HSTS and `proxy_cookie_flags session_id samesite=lax secure`.
- Serve `/{module}/static/**` from the proxy (add `Content-Security-Policy: default-src 'none'` on images); WSGI hosting exists but needs a separate `--no-http` server for cron and a gevent server for livechat.
- Public-server hygiene: never install demo data, no `admin` login, no default passwords, SSH keys only, fail2ban on login logs, daily off-site backups of db + filestore, prefer Linux.

### Windows Source-Run Pitfalls (battle-tested)

- Odoo aborts at startup when `db_user` is `postgres` — create a dedicated role with CREATEDB (e.g. `CREATE ROLE wms_odoo LOGIN PASSWORD '...' CREATEDB;`).
- Windows PostgreSQL refuses `CREATE DATABASE ... LC_COLLATE 'C'` (odoo/modules/db.py hardcodes it with template0) — pre-create the database with cluster-default collation, plus `CREATE EXTENSION pg_trgm` as superuser.
- `configparser` reads odoo.conf in the locale encoding (GBK on zh-CN) — keep the conf pure ASCII (an ASCII junction like `E:\odoo-proj -> <project dir>` makes every path ASCII).
- Missing `USERPROFILE` makes config defaults crash in appdirs (`SHGetFolderPath` COM error) — set `USERPROFILE`/`APPDATA`/`HOMEDRIVE`/`HOMEPATH` in the launcher.
- PG18 removed the `lc_collate`/`lc_ctype` GUCs (locale is per-database now).
- Windows: multi-threaded server only (`workers > 0` is Linux-only).

### --dev=assets is a per-page-load full bundle rebuild (measured: 18s on Windows)

- Log evidence: `Generating a new asset bundle attachment ...web.assets_web.min.js ... 18.447 rw` (3.1 MB) plus 6-7s per CSS bundle. With `--dev=assets` this happens on page loads during frontend iteration - the whole web client feels sluggish.
- Keep `--dev=reload,xml,qweb` in the daily launcher. Enable `assets` ONLY while actively editing JS/OWL/SCSS, and remove it after. One-time alternatives: append `?debug=assets` to the URL once, or rely on `-u` upgrade + hard refresh (Ctrl+F5) to pick up new static files into the cached bundle.

### Blank webclient after static-file fixes: stale cached asset bundles

- A bundle built while a static file was broken (e.g. invalid XML in an OWL template) stays cached as an `ir.attachment`; the bundle URL hash can survive content fixes, so the client keeps boot-crashing on the poisoned bundle (blank page, and zero `/web/assets/` GETs in the server log because the browser cache/service worker serves it).
- Fix server-side: delete the cached bundles (`ir.attachment` with `url like '/web/assets/%'`) + clear registry caches; they regenerate on the next page load (expect one slow load). Client-side: hard refresh twice; if still blank, unregister the service worker / clear site data in DevTools.

### One-shot restart script: kill port, upgrade fail-fast, detached start, health poll

- Codify the agent's post-change ritual into a ps1: (1) Stop-Process on the listener found via Get-NetTCPConnection -LocalPort; (2) optional `-u` with `--stop-after-init` plus tests, aborting on $LASTEXITCODE so a broken module never starts; (3) launch via Start-Process -WindowStyle Hidden - the detached child SURVIVES the console exit (a run_in_background pwsh job holding the server does not); (4) poll /web/login until 200.
- This lets the AI agent own restarts after each change round while the user only refreshes the browser; give start-wms.bat a netstat/taskkill guard so both entry points stay conflict-free.