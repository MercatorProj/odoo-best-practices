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

Note: In Odoo 17+, `<tree>` was renamed to `<list>`. Chatter uses `<chatter/>` instead of `<div class="oe_chatter">` with separate message and followers fields.

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
```
