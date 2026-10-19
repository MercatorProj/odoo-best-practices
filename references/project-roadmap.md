# Project Roadmap: Real-World Odoo Delivery

A phase-by-phase working route for real Odoo projects (implementation, customization,
or productization). Order follows hard dependencies; every feature later re-runs the
same micro-loop (model -> logic -> test -> i18n -> version bump).

## Phase 0 - Environment & Engineering Baseline

**Do**
- Keep the Odoo source locally searchable - debugging always starts from reading the
  standard implementation, never from guessing.
- One-shot script first: kill process -> `odoo -u <module> --test-enable --stop-after-init`
  -> start server, with logs to a known file. This script becomes the quality gate.
- Module skeleton: treat the manifest version as the release cadence (bump per feature
  or hotfix); the `data` list order IS the dependency declaration (reports load before
  views that reference their XML ids).

**Gate:** one command performs upgrade + tests + restart; anyone can run the project.

**Pitfall:** skipping the one-shot script costs ~5x iteration speed for the whole project.

## Phase 1 - Master Data Modeling

**Do**
- Model along the dependency chain: organization/locations -> categories/config tables ->
  core master data.
- Uniqueness as `UNIQUE(field, company_id)`; trigram indexes for barcode/name search.
- `company_id` always a related field up the chain plus `_check_company_auto = True`.
- Deletion guards with `@api.ondelete` (has-children, done-documents); attribution
  fields (`*_by`, dates) decided at modeling time, not retrofitted.

**Gate:** a reviewed model diagram where every table answers: who owns it, who references
it, what breaks when it is deleted.

**Principle:** configuration lives on the parent (category/config table); records inherit
it via `related + store` read-only - a single source of truth.

## Phase 2 - Business Documents & State Machines

**Do**
- Draw the state machine BEFORE coding: states, legal transitions, and which migration
  carries which side effect (confirm creates/reserves, done posts).
- Separate an append-only ledger (immutable once done) from mutable on-hand state;
  guard quantities with DB-level `CHECK`.
- Design rollback semantics for cancellable documents (release reservations, remove
  draft artifacts).

**Gate:** every transition has a test; illegal transitions raise `UserError` with a
human-readable message.

**Principle:** side effects live only in state-transition methods; UI calls methods,
never writes state fields directly.

## Phase 3 - Interaction Layer

**Do**
- Climb the cost ladder: form button/onchange -> wizard -> dedicated OWL client action.
- Wrap complex interactions as model-level APIs (methods returning plain dicts) so the
  same semantics serve the form, RPC callers, and batch scripts.
- Progressive enhancement: keyboard/scanner first, mouse optional.

**Gate:** every interaction path passes a real-browser check (DOM assertions + zero
console errors).

**Iron rule:** green backend tests do NOT mean the UI works - a live browser run of the
user path (login -> deep link -> assert DOM, collect console) is mandatory for any
client-action change.

**OWL2 / client-action hard rules** (each proven by a live crash):
- `ir.actions.client` `params` is a PYTHON literal (safe_eval): `True`, not `true`.
- Declare `props = useProps()` as a class field or `this.props` is undefined.
- Template expressions run in a strict scope: instance members need `this.` prefixes,
  loop/t-set vars stay bare, globals like `document` are invisible - expose via getters.
- Raw `&&` inside a template attribute is an XML entity error (`&amp;&amp;`); use
  `t-out` (`t-esc` is deprecated).
- The bundled owl has no DOM-ref hook and bare `t-ref` throws unless refs are wired -
  prefer querying a unique overlay class.

## Phase 4 - Reporting & Integration

**Do**
- QWeb reports: HTML first (`qweb-html`), switch `report_type` to PDF once the engine
  exists - one-line change later.
- Report XML loads BEFORE views referencing its action xmlid.
- External APIs go through model methods with the same validation/ACLs as internal use.

## Phase 5 - Quality Gates (cross-cutting, every feature)

| Gate | Content |
|---|---|
| Tests | register new files in `tests/__init__.py` (silent no-run otherwise); state machines, constraints, guards each covered |
| UI | real-browser verification with zero console errors |
| i18n | English source strings -> po pipeline -> zero untranslated |
| Version | bump per feature/hotfix; the version IS the traceable release unit |

## Phase 6 - Release & Operations

- Traceable artifacts: repository tag + Release + attached package.
- Rehearse upgrades on a database WITH legacy data: a new `required` field on a
  non-empty table gets NO NOT NULL constraint, and `@api.constrains` does not fire
  when the field is absent from vals - enforce with a `create`/`write` override.
- Pre-go-live: HTTPS (hard requirement for camera/geolocation APIs), backups, log rotation.

## Upfront Decision Checklist (answer before coding)

1. Tracking granularity: by quantity or by individual piece? -> decides the dual-line model.
2. Which state transitions are irreversible? -> decides guards and rollback design.
3. Who operates on which device? -> decides interaction shape (desktop form vs PDA
   fullscreen vs gun scanner).
4. Multi-company / multi-language from day one? -> related company + po pipeline are
   painful to retrofit.
5. Where does knowledge settle? -> write every pitfall into the team's skill/reference
   library; a project should leave behind methodology, not chat logs.

## Per-Feature Micro-Loop

```
model -> logic -> tests -> po -> version bump -> gates -> (publish)
```

Run it per feature; the phases above only order the first pass through it.
