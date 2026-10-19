# Views and OWL Components

## XML View Structure

### Form View

```xml
<record id="my_model_form" model="ir.ui.view">
    <field name="name">my.model.form</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <form>
            <header>
                <field name="state" widget="statusbar" statusbar_visible="draft,confirmed"/>
                <button name="action_confirm" type="object" string="Confirm" class="btn-primary" invisible="state != 'draft'"/>
            </header>
            <sheet>
                <widget name="web_ribbon" title="Archived" bg_color="text-bg-danger" invisible="active"/>
                <group>
                    <group>
                        <field name="name"/>
                        <field name="partner_id"/>
                    </group>
                    <group>
                        <field name="company_id" groups="base.group_multi_company"/>
                    </group>
                </group>
                <notebook>
                    <page string="Lines" name="lines">
                        <field name="line_ids">
                            <list editable="bottom">
                                <field name="product_id"/>
                                <field name="quantity"/>
                            </list>
                        </field>
                    </page>
                </notebook>
            </sheet>
            <chatter/>
        </form>
    </field>
</record>
```

Note: In Odoo 17+, `<tree>` was renamed to `<list>`. Chatter uses `<chatter/>` instead of `<div class="oe_chatter">` with separate message and followers fields; the element requires the model to inherit the `mail.thread` mixin.

### List View

```xml
<record id="my_model_list" model="ir.ui.view">
    <field name="name">my.model.list</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <list>
            <field name="name"/>
            <field name="partner_id"/>
            <field name="amount_total" sum="Total"/>
            <field name="state" widget="badge" decoration-info="state == 'draft'" decoration-success="state == 'done'"/>
        </list>
    </field>
</record>
```

### Search View

```xml
<record id="my_model_search" model="ir.ui.view">
    <field name="name">my.model.search</field>
    <field name="model">my.model</field>
    <field name="arch" type="xml">
        <search>
            <field name="name" string="Name or Ref"/>
            <field name="partner_id"/>
            <filter name="draft" string="Draft" domain="[('state', '=', 'draft')]"/>
            <group expand="0" string="Group By">
                <filter name="group_state" string="State" domain="[]" context="{'group_by': 'state'}"/>
            </group>
        </search>
    </field>
</record>
```

## View Inheritance

Use `xpath` to locate elements. Use `position="inside"`, `position="before"`, `position="after"`, `position="replace"`, or `position="attributes"`.
- `expr` must match **exactly one** element — it raises an error on zero or multiple matches.
- In `replace` position, a `$0` node in the new body keeps the original element.

```xml
<record id="my_model_form_inherit" model="ir.ui.view">
    <field name="name">my.model.form.inherit</field>
    <field name="model">my.model</field>
    <field name="inherit_id" ref="other_module.original_form"/>
    <field name="arch" type="xml">
        <xpath expr="//field[@name='partner_id']" position="after">
            <field name="custom_field"/>
        </xpath>
    </field>
</record>
```

## OWL 2 Components

For custom frontend behaviour, create an OWL component with a JS file, an XML template file, and optionally a SCSS file, all loaded via the manifest's `assets` bundle.

Key conventions:

- Use `setup()` for initialisation; never override `constructor()`.
- Template naming: `addon_name.ComponentName` (e.g., `mycompany_feature.MyWidget`).
- Define templates in XML files (not inline `xml` tagged literals) so they can be translated.
- Template files use `<templates xml:space="preserve">` as the root element.

```javascript
/** @odoo-module **/

import { Component, useState } from "@odoo/owl";
import { registry } from "@web/core/registry";

export class MyWidget extends Component {
    static template = "mycompany_feature.MyWidget";

    setup() {
        this.state = useState({ value: 1 });
    }

    increment() {
        this.state.value += 1;
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<templates xml:space="preserve">
    <t t-name="mycompany_feature.MyWidget">
        <div t-on-click="increment" class="my_widget">
            <span t-esc="state.value"/>
        </div>
    </t>
</templates>
```

Key OWL 2 concepts:

- `useState` for reactive state that triggers re-render.
- `onWillStart` for async setup before the first render.
- `t-on-event` for event handlers; `t-esc` for text output; `t-out` for HTML output.
- Use `registry` to register custom fields, actions, and services.
- Templates live in static XML files loaded via the `assets` bundle in the manifest.

Reusable built-in components (all from `@web/core/`): `ActionSwiper`, `CheckBox`, `ColorList`, `Dropdown` / `DropdownItem`, `Notebook`, `Pager`, `SelectMenu`, `TagsList`.

## Kanban Views and QWeb Basics

- Kanban is a QWeb-powered card view (root `<kanban>`); add `kanban` to the action's `view_mode`.
- The `<templates>` block must define a root template named `card`, rendered once per record; `<t t-name="card">` is the QWeb placeholder.
- QWeb directive `t-if` renders an element conditionally; `t-foreach`/`t-out` cover loops and output.
- Inside templates, `record` exposes each field as `record.<name>.value` (formatted per user locale) and `record.<name>.raw_value` (raw `read()` value).
- Fields needed for expressions but not displayed go outside `<templates>` (e.g. `<field name="state"/>`).
- `default_group_by="<field>"` groups records by default when the action/search specifies none.

OWL essentials from the JS framework tutorial:

- Props (parent -> child) are validated with `static props = { title: String, ... }` in dev mode; make it a habit for every component.
- Callback props (child -> parent) pass a function down; in templates give it with the `.bind` suffix (e.g. `onChange.bind`).
- Slots make components generic: the default slot renders whatever content the parent passes.
- `t-foreach` in OWL **requires a unique `t-key`** (unlike the Python QWeb implementation).
- Dynamic attributes: `t-att-class` combines with static `class`; falsy expressions emit no attribute.
- DOM access: `t-ref="name"` + `useRef("name")` — `.el` exists only after mount; name refs with a `Ref` suffix.
- Lifecycle: setup -> onWillStart -> mount -> onMounted; updates via onWillUpdateProps/onWillPatch/onPatched; teardown via onWillUnmount/onWillDestroy. **Hooks may only be called inside `setup()`**; names start with `use` or `on`.
- HTML safety: `t-esc` always escapes (and is slightly faster); `t-out` renders HTML only for values wrapped with `markup()`.

OWL in the web client (dashboard tutorial):

- **Services** are persistent singletons (component state dies on unmount): register with `registry.category("services").add(name, { dependencies: [], start(env, deps) { return api; } })`, consume with `useService(name)` from `@web/core/utils/hooks`.
- `action` service: `useService("action")` then `doAction(xmlIdOrDict)` opens any standard or dynamic action.
- Server calls: `import { rpc } from "@web/core/network/rpc"` and `await rpc(route, params)`, typically inside `onWillStart(async () => ...)`.
- Cache expensive calls with a service + `memoize` from `@web/core/utils/functions`; share live data by returning a `reactive` object from the service and subscribing with `useState(service.reactive)` — mutations update every subscriber.
- Dynamic components: `<t t-foreach="items" t-as="item" t-key="item.id"><t t-component="item.Component" t-props="itemProp"/></t>`.
- Lazy-load heavy assets with `LazyComponent` from `@web/core/assets`: register the real component in the `lazy_components` registry and an intermediate loader in `actions`.
- Extensibility pattern: expose your own registry category (e.g. `awesome_dashboard`) so other addons can contribute items.

## Custom HTML/JS and Third-Party Libraries

- Ship JS/CSS via the manifest: `'assets': {'web.assets_backend': ['my_module/static/src/**/*']}` — files become importable ES modules.
- Register custom components where they belong: `registry.category("fields")` for field widgets (`<field widget="my_widget"/>`), `registry.category("actions")` for client actions, or your own registry category for pluggable items.
- Wrapping a third-party JS library (the Chart.js pattern): declare the lib in assets **or** lazy-load with `loadJs`; in the component, grab the DOM node with `t-ref` + `useRef`, initialise the lib inside `onMounted`, and destroy it in `onWillUnmount` to avoid leaks.
- Raw HTML inside templates is escaped by default; only values wrapped with `markup()` render as HTML via `t-out`. HTML content lives in `fields.Html`.
- Embedding external pages works as plain HTML (`<iframe t-att-src="..."/>`) — visibility is governed by the target site's X-Frame-Options/CSP.

## Domain Syntax in XML

A domain is a list of triplets `(field, operator, value)`. Criteria combine with an implicit AND; explicit operators `&` (AND), `|` (OR), `!` (NOT) are used in **prefix position**:

```xml
<!-- services OR unit_price NOT in [1000, 2000) -->
<field name="domain">['|', ('product_type', '=', 'service'),
    '!', '&', ('unit_price', '>=', 1000), ('unit_price', '<', 2000)]</field>
```

XML forbids raw `<` and `&` inside attributes — use `&lt;` and `&amp;`.

Search views use `<filter>` toggles with `domain` and/or `context` (key `group_by` groups results):

```xml
<filter name="available" string="Available"
        domain="['|', ('state', '=', 'new'), ('state', '=', 'offer_received')]"/>
<filter name="group_postcode" string="Postcode" context="{'group_by': 'postcode'}"/>
```

## Buttons in Views

- `type="object"`: `name` is a **public** Python method (no `_` prefix) executed on the recordset.
- `type="action"`: `name="%(module.xml_id)d"` triggers a window action by external id.
- Form headers: `<header>` holds state buttons plus `<field name="state" widget="statusbar"/>`; list rows take icon buttons (`icon="fa-check"`).

## PDF Reports

- Layout: a top-level `report/` folder — QWeb templates in `*_templates.xml`, `ir.actions.report` actions in `*_reports.xml`; register both in the manifest `data` list (loaded sequentially, templates before actions that reference them).
- Template skeleton: `<template id="report_x">` looping `docs` (`t-foreach="docs" t-as="rec"`), then `t-call="web.html_container"` and `t-call="web.external_layout"` (company header/footer) around a `<div class="page">`. Render values with `t-field`; Bootstrap and Font Awesome classes work.
- `binding_model_id` on the action puts the report in the model's Print menu; multi-select in a list view produces one combined PDF (the template loops the whole recordset). Reports for specific states go on a button instead.
- Sub-templates: extract shared parts and call them with `t-call`; cross-module bindings reference the comodel as `ref="base.model_res_users"`.
- Inherit a report with `inherit_id="module.template_id"` + the same xpath machinery as view inheritance.
- Gotchas: stray spaces inside text nodes break report translations; reports are also web pages (URL-viewable, security still enforced); barcodes are built in.

## Pipeline UI Toolbox

- Form header progress strip: `widget="statusbar"` works for both Selection states and Many2one stages. For stages, take options from the crm reference implementation: `options="{'clickable': 1, 'fold_field': 'fold'}"` plus `statusbar_visible` for the always-shown prefix.
- Clicking a statusbar segment writes the field from the UI — server-side `write()` guards remain the real enforcement (view is not security).
- Kanban drag & drop between grouped columns is built-in for many2one groupbys (source: `kanban_renderer.js` `DRAGGABLE_GROUP_TYPES = ["many2one"]`); no kanban attribute disables it in the 20.0 view docs (`disable_drag_drop` is gantt-only).
- Kanban column summary bars: `<progressbar field="..." colors="{...}" sum_field="..."/>`.
- List rows: `decoration-*` attributes and `widget="badge"` color-code the current stage.
- Follow-ups: the `mail.activity.mixin` adds schedulable activities surfaced on kanban and form.

## Native Field Widget Catalog

- The complete native field widget catalog lives in `reference/frontend/javascript_reference` (20.0): **51 widgets** — 33 non-relational (`integer`, `float`, `monetary`, `date`, `daterange`, `relative_date`, `phone`, `email`, `url`, `domain`, `priority`, `state_selection`, `boolean_toggle`, `percentpie`, `progressbar`, `statinfo`, `dashboard_graph`, `ace`, `badge`, `image`, `attachment_image`, `handle`, ...), 16 relational (`many2one` + barcode/avatar variants, `many2many_tags` + color-dot/avatar variants, `many2many_checkboxes`, `radio`, `selection`, `one2many`, `statusbar`, `reference`), and 2 standalone (`web_ribbon`, `week_days`).
- Each entry documents supported field types, attributes (`digits`, `options`, `placeholder`) and options; check it before writing a custom widget.

## List View Polish

- `editable="bottom"` lets users edit records directly in the list; use when a model has few fields (e.g. tags, offer lines).
- `optional="show"` / `optional="hide"` on a field makes it user-toggleable and controls its default visibility.
- Decorations color rows: `decoration-success|info|warning|danger|muted="<python expr>"`.
- **Every field used in a view expression (invisible, readonly, required, decoration-*) must be present in the view** (use `invisible="1"` to keep it hidden).
- Add a field to a list or form view **only once** — duplicates are not supported.

## Conditional Attributes

- `invisible`, `readonly` and `required` accept arbitrary Python expressions (17+ syntax, replaces `attrs`): `invisible="not garden"`, `invisible="state in ('sold', 'cancelled')"`.
- They apply to fields, buttons and groups alike.
- ⚠️ View-level `readonly` is UX only — the server performs no check and the field stays writable over RPC. Enforce with ACLs, record rules, field-level `readonly=True`, or constraints.

## Inline Views

- Nest a `<list>` directly inside a One2many field to show a context-specific column set:

```xml
<field name="property_ids">
    <list>
        <field name="name"/>
        <field name="expected_price"/>
    </list>
</field>
```

## Search Tweaks

- Activate a filter by default via the action context: `context="{'search_default_available': 1}"`.
- `<field name="living_area" filter_domain="[('living_area', '>=', self)]"/>` — `self` is the user input; the example searches 'at least'.

## Stat Buttons

- `<button class="oe_stat_button" type="action" name="%(module.action_xmlid)d" icon="fa-tag"><field name="offer_count" widget="statinfo" string="Offers"/></button>`.
- Filter the target action with a domain using `active_id`: `domain="[('property_type_id', '=', active_id)]"`.
- Counters typically use a computed field over the One2many inverse of a **stored related** field.

## Actions and Menus

```xml
<record id="my_model_action" model="ir.actions.act_window">
    <field name="name">My Model</field>
    <field name="res_model">my.model</field>
    <field name="view_mode">list,form</field>
    <field name="context">{'search_default_draft': 1}</field>
    <field name="help" type="html"><p>No records yet.</p></field>
</record>

<menuitem id="my_model_menu" name="My Model" parent="parent_menu_id" action="my_model_action" sequence="10"/>

Menus follow a three-level architecture: root menu (app switcher) > first-level menu (top bar) > action menu. Define them nested in one XML block; the action menu takes its name from the linked action:

```xml
<menuitem id="my_app_menu_root" name="My App">
    <menuitem id="my_app_first_level_menu" name="Records">
        <menuitem id="my_model_menu_action" action="my_model_action"/>
    </menuitem>
</menuitem>
```

After adding menus or views, refresh the browser: the web client caches menus and views.
```

## XML ID Naming Conventions

From the official coding guidelines ("XML IDs and naming"). Keep `name` identical to the xml id with dots replacing underscores:

- Menu: `<model_name>_menu` (root often `<model_name>_menu_root`); submenus `<model_name>_menu_<do_stuff>`.
- View: `<model_name>_view_<view_type>` (form, list, kanban, search, ...).
- Action: main `<model_name>_action`; extra ones `<model_name>_action_<detail>`; window actions may be `<model_name>_action_view_<view_type>`.
- Group: `<module_name>_group_<group_name>` (user, manager, ...).
- Rule: `<model_name>_rule_<concerned_group>` (user, public, company, ...).
- Prefer the shortcut tags (`<menuitem>`, `<template>`) over raw `<record>` notation where available.

### OWL 20.0 component API: useState/useRef removed

- Verified in odoo 20.0: `@odoo/owl` no longer exports useState or useRef (owl.js has zero occurrences). Official components create reactive state with `this.state = proxy({...})` (89 occurrences in web/static/src) importing `proxy` from "@odoo/owl".
- Template refs changed too: the heavy official form is `xxxRef = useProps.static("name", t.signal(t.ref()).optional(() => signal.ref()))` with `t-ref="this.xxxRef"` and `this.xxxRef().value`. For simple cases (focus/read an input inside an event handler) prefer `ev.target` and a defensive `this.el.querySelector(...)` on mount - no refs needed.
- Failure signature when using the removed API: `Uncaught TypeError: useState is not a function at Component.setup` inside the bundle - the module loads fine, it dies at setup time.
- Asset cache reminder: bundle URL hashes are content-independent; after ANY static JS/XML/SCSS fix, delete ir.attachment rows with url like /web/assets/% and hard-refresh (or restart) to force a rebuild.
- 20.0 search views: the `<group>` group-by section is gone - the Group By menu auto-generates from stored fields. Keep `<search>` to `<field>` + `<separator>` + `<filter domain=...>` only; an `expand` attr (or any `<group>`) fails RELAXNG validation and blocks module upgrade.
- Operator/kiosk pattern: an `ir.actions.client` with `target: fullscreen` hides the navbar natively (ActionMode supports it); one component renders two screens (pick list + work) switched by state. `requestFullscreen()` needs a user gesture - call it inside click handlers, never on load. Toggle-mode chrome changes (body class CSS) MUST be cleaned up `onWillDestroy` or the hidden navbar leaks to other screens. Scan-first navigation: the same input routes by code prefix (`OUT/`|`IN/` -> resolve+switch delivery, else scan item).
- `ir.actions.client` `params` holds a PYTHON literal safe_eval'd at read time (base ir_actions `_compute_params`), NOT JSON - a JSON `true` crashes menu clicks with `NameError: true`; write `True`. Regression-test action records by reading `.params` in tests (it triggers the same eval path). Verify features through the exact RPC path users hit (`/web/action/load` type=json, action_id in body; `ir.ui.menu.action` returns a `model,id` reference string, parse the int out) - not adjacent surface like page HTML or bundle content.
- OWL 2 (Odoo 20) hard rules, each proven by a live crash: (1) `this.props` does NOT exist unless the component declares the class field `props = useProps();` - accessing it anyway throws at the first reader. (2) Template expressions run in a STRICT scope: instance members need explicit `this.` (`this.state.x`, `this.labels.x`), loop/t-set vars stay bare, and globals like `document` are INVISIBLE - expose them through component getters (e.g. `get isFullscreen()`). (3) Raw `&&` inside a template attribute is an XML entity error (`xmlParseEntityRef`) - write `&amp;&amp;`; `t-esc` is deprecated, use `t-out`. (4) A setup() that throws mid-way leaves a half-initialized component whose template then crashes with a misleading error - when a template reports `undefined`, suspect an earlier silent setup failure first.
