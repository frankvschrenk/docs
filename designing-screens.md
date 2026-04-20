---
layout: default
title: Designing screens
nav_order: 6
---

# Designing screens

A **screen** is how a context is rendered on the desktop. Every screen
is an XML file stored in `oos.dsl`. When the client opens a context
— because the user clicked a list entry, typed a query the assistant
resolved, or the AI pushed a board — it fetches the screen, wires it
up to the data envelope from oosp, and hands the whole thing to a Fyne
builder.

This page is a tour of what you can put in a screen file. The working
examples live alongside the demo seed and cover forms, tables, tabs,
accordions, dropdowns, sliders and theme attributes.

## The skeleton

```xml
<screen id="person_detail" title="Person — Detail"
        delete="true" save="true" exit="true" cur="EUR" locale="de-DE">
  <box orient="horizontal" p="3">
    <icon name="account" size="48"/>
    <box orient="vertical" ml="3" expand="true">
      <richtext>
        <span style="heading">Person — Detail</span>
        <span style="bold">Employee profile</span>
      </richtext>
    </box>
  </box>
  <sep/>

  <tabs p="2">
    <tab label="Details">
      <section label="Personal data" p="3">
        <field label="First name" bind="person.firstname" focus="true"/>
        <field label="Last name"  bind="person.lastname"/>
      </section>
    </tab>
  </tabs>
</screen>
```

## Screen attributes

- `id` — matches the context name. This is how the client finds the
  screen for a given context.
- `title` — what appears in the window title bar.
- `delete`, `save`, `exit` — whether the corresponding header buttons
  are rendered. The delete button is also subject to the role's
  permission to delete.
- `cur`, `locale` — number and currency formatting defaults for the
  entire screen. Fields can override them.
- `scroll="true"` — wraps the screen body in a vertical scroll
  container. Use for long detail forms; the default for list screens
  is no scroll because tables manage their own scrolling.
- `label-color="primary"` — sets the colour of section labels for the
  whole screen.

## The important widgets

### Layout

- `<box>` — a horizontal or vertical stack, set via `orient`.
- `<section>` — a labelled region with its own column layout.
- `<grid cols="2">` — explicit column grid.
- `<tabs>` / `<tab>` — a tab group.
- `<accordion>` / `<accordion-item>` — collapsible regions; one can
  be `open="true"`.
- `<card>` — a bordered panel with a title and subtitle.
- `<sep/>` — a separator line.

All of these take optional spacing attributes: `p`, `pt`, `pb`, `pl`,
`pr`, `mt`, `mb`, `ml`, `mr`, `gap`. Numeric units, multiplied by
the theme's base spacing.

### Data entry

- `<field label="..." bind="entity.field"/>` — the everyday field.
  Defaults to an entry widget; use `widget="..."` to switch.
- `<field widget="textarea"/>` — multiline.
- `<field widget="choices" options="roles"/>` — dropdown backed by a
  meta list.
- `<field widget="radio"/>` — radio group.
- `<field widget="check" label="Active"/>` — checkbox.
- `<field widget="slider" min="0" max="100" step="1"/>` — slider.
- `<field widget="progress"/>` — readonly progress bar bound to a
  float value between 0 and 1.

`readonly="true"` disables editing. `focus="true"` on exactly one
field causes the window to give it initial focus.

### Data display

- `<table bind="rows" action="on_select">` — the main collection
  widget. Columns are declared as children:
  ```xml
  <column field="firstname" label="First name" width="140"/>
  <column field="age" label="Age" width="70" format="num:0"/>
  ```
- `<list>` — a single-column list.
- `<tree>` — a hierarchical list.
- `<label text="..."/>` — a plain label.
- `<richtext>` with `<span style="...">` — rich text with styled
  segments. Also supports Markdown when `markdown="true"`.
- `<hyperlink text="..." url="..."/>` — opens in the system browser.

### Actions

- `<button action="on_new" label="New" style="add"/>` — toolbar or
  inline button. `action` must match a `<navigate>` or `<action>`
  event on the corresponding context.
- `<toolbar>` — a row of buttons and separators.

## Bindings

A `bind` attribute connects a widget to the state. The state comes
from the envelope that oosp returns: a content section (the record)
and a meta section (the dropdown sources).

Paths use dot notation:

- `person.firstname` — a field inside the `person` content.
- `rows` — the rows array for a table or list.
- `selected.id` — the id of the currently selected row, written back
  by the table widget when the user picks a row.

## Formatting

- `format="num:0"` — fixed-point number with no decimals.
- `format="@"` — currency, using the screen's `cur` attribute.
- `format="datetime:short"` — short locale-aware datetime.

## Developing screens

The **ooso** tool ships a live preview. You load a screen file, feed
it a JSON envelope with sample content and meta, and see the rendered
Fyne UI update as you edit. This is the fastest loop; the alternative
is to save the screen into `oos.dsl`, rebuild the client, and
navigate to the context — which works, but takes longer per
iteration.

Read [The toolbox](./the-toolbox.html) for what ooso does and how to
use it.

## What's next

- [The AI assistant](./the-ai-assistant.html) — how the LLM consumes
  your context and screen definitions.
- [Security and roles](./security-and-roles.html) — what the screen's
  buttons respect.
