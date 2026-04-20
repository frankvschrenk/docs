---
layout: default
title: Writing contexts
nav_order: 5
---

# Writing contexts

A **context** is OOS's name for an addressable view onto your data. A
context is either a *collection* — a list of records — or an *entity*
— a single record with all its fields. Each context lives in
`oos.ctx` as a row of XML. From this XML, OOS builds the GraphQL
schema, the permission matrix, and the AI-facing schema chunks.

This page is the reference for what you can put in a context file.
For a worked example, read the demo context under `oos-run/seed` —
it ships alongside the seed data and covers most of the features.

## The skeleton

```xml
<oos>
  <context name="person_list" kind="collection" source="person" dsn="demo">
    <permission role="admin"   actions="read,write,delete"/>
    <permission role="manager" actions="read,write"/>
    <permission role="user"    actions="read"/>

    <list_fields>id, firstname, lastname, email, city, age</list_fields>

    <field name="id"        type="int"    readonly="true"/>
    <field name="firstname" type="string" filterable="true"/>
    <field name="lastname"  type="string" filterable="true"/>
    <field name="age"       type="int"    filterable="true"/>
    <!-- ... -->

    <navigate event="on_select" to="person_detail" bind="id -> id"/>
    <navigate event="on_new"    to="person_detail" bind=""/>

    <relation name="notes" context="note_list" type="has_many" bind="id -> person_id"/>
  </context>
</oos>
```

## Attributes on the context element

| Attribute | Meaning |
| --- | --- |
| `name`   | The identifier used everywhere. Conventionally ends in `_list` for collections and `_detail` for entities. |
| `kind`   | `collection` or `entity`. Collections show a list; entities show one record. |
| `source` | The underlying database table. Multiple contexts may share the same source. |
| `dsn`    | Which configured data source to read from. Defined in `setup.toml`. |

## Permissions

Each `<permission>` line declares which actions a role may perform on
this context. Three actions exist: `read`, `write`, `delete`. A
context with no `<permission>` entries at all is treated as
unrestricted — every role may do everything. Once one entry is
present, the context is in deny-by-default mode for every other role.

Contexts that share a table do not share permissions: you can have
one context for admins with full access and a second one for ordinary
users that exposes only a read-safe subset.

## Fields

Every field has at least a `name` and a `type`. Supported types:
`int`, `float`, `string`, `text`, `bool`, `datetime`.

Useful optional attributes:

- `readonly="true"` — the field is shown but not editable. Always set
  this on `id`, `uuid`, `created_at`, `updated_at`.
- `filterable="true"` — the field becomes a GraphQL filter argument.
  Strings gain a `_like` suffix variant, numbers gain `_gt` and `_lt`.
- `meta="roles"` — the field's values come from a `<meta>` reference
  list rather than from free-text input.

Filter examples tell the AI assistant how a filter *should* look:

```xml
<field name="age" type="int" filterable="true">
  <example op="gt" value="50">Personen älter als 50</example>
  <example op="lt" value="30">Personen jünger als 30</example>
</field>
```

These examples land in the schema chunk and give the assistant a
verbatim shape to copy.

## Meta references

A `<meta>` element declares a reference list for dropdown fields:

```xml
<meta name="roles"
      table="role"
      value="key"
      label="label"
      dsn="demo"
      order_by="label"/>
```

At runtime, oosp generates a GraphQL query named `meta_roles` that
returns `{ value, label }` pairs from the `role` table. Any field
with `meta="roles"` on an entity context will have its dropdown
populated from that query. The client fetches the meta queries
alongside the record itself in a single envelope.

## Navigation

Navigation moves the user between contexts. Two common patterns:

```xml
<navigate event="on_select" to="person_detail" bind="id -> id"/>
<navigate event="on_new"    to="person_detail" bind=""/>
```

- `on_select` fires when the user picks a row in a collection. The
  `bind` maps the selected row's `id` to the detail's `id` argument.
- `on_new` fires when the "new" button in a collection is clicked.
  Empty `bind` means "open the detail screen with no record."

## Relations

```xml
<relation name="notes" context="note_list" type="has_many" bind="id -> person_id"/>
```

Relations describe the shape of the graph. The AI assistant uses them
to understand that notes belong to a person, so "show me Anna's notes"
translates to `note_list(person_id: <Anna's id>)`.

## AI hints

You can add `<ai name="...">` blocks anywhere inside a context to
give the assistant extra prose context. These are rendered into the
schema chunk as free-form notes:

```xml
<ai name="scope">
  Persons are employees, customers and contacts. One row per person.
</ai>
<ai name="format_hint">
  net_worth is an amount in EUR. age is in years, no decimals.
</ai>
```

Use them sparingly. They are free-form, which means the assistant
has to parse them with its prose understanding — costly if they are
long, not always reliable. Reserve them for things that are genuinely
hard to express elsewhere.

## Actions

Actions are write operations that the entity screen offers:

```xml
<action event="on_delete" type="delete" confirm="Person wirklich löschen?"/>
<action event="save"      type="save"/>
```

Two types are supported: `save` and `delete`. The `confirm` attribute
makes the client prompt before executing; omit it for actions that
should fire immediately.

## What's next

- [Designing screens](./designing-screens.html) — now that your data is
  described, lay it out.
- [Security and roles](./security-and-roles.html) — the full picture of
  how permissions flow through the system.
