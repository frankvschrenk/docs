---
layout: default
title: The DSL
nav_order: 4
---

# The DSL

OOS uses two small declarative languages: the **domain DSL** (`.domain`)
and the **view DSL** (`.view`). Domains describe data; views describe
how data is presented. Both are edited in oosd with syntax highlighting,
live diagnostics and autocompletion.

## Domain DSL

A domain file describes one entity — its database table, its fields,
its permissions, its relations to other domains, its dropdown sources,
and hints for the AI assistant.

### Basic structure

```
domain <name> from <table>@<datasource> {
  permission <role>  <actions>
  field <name> : <type> [readonly] [filterable] { ... }
  relation <name> : <kind> <target> bind=<local> -> <foreign>
  meta <name> from <table> <key-col> <label-col> [order_by <col>]
  ai "<tag>" "<text>"
  aliases [ "...", "..." ]
}
```

### Field types

| Type | Description |
|---|---|
| `int` | Integer |
| `float` | Floating-point number |
| `string` | Short text |
| `text` | Long text |
| `bool` | Boolean |
| `datetime` | Timestamp |

### Field modifiers

- `readonly` — the field cannot be mutated via the AI or the UI.
- `filterable` — the AI may use this field as a filter.
- `options=<meta>` — the field is a dropdown backed by a meta source.

### Filter examples

Filterable fields can carry examples that the AI uses to understand
the expected filter shape:

```
field age : int filterable {
  example gt 50 "persons older than 50"
  example lt 30 "persons younger than 30"
}
```

Supported operators: `eq`, `neq`, `like`, `gt`, `gte`, `lt`, `lte`.

### Relations

```
relation notes : has_many note bind=id -> person_id
```

This declares that a `person` has many `note` records, joined on
`person.id = note.person_id`.

Supported relation kinds: `has_many`, `belongs_to`.

### Meta sources

Meta sources back dropdown fields. They point to a lookup table and
declare which column is the key and which is the label:

```
meta cities from city name name order_by name
```

### AI hints

AI hints are free-form instructions that are embedded alongside the
field descriptions. Use them to clarify scope, format rules or
edge cases:

```
ai "scope"
   "Persons are employees, customers and contacts. One row per person."

ai "edit_behavior"
   "id, uuid, source, created_at and updated_at are readonly."
```

### Aliases

Aliases are synonyms for the domain name, used during semantic
retrieval:

```
aliases [ "mitarbeiter", "kunden", "kontakte" ]
```

### Full example

```
domain person from person@demo {

  permission admin   read, write, delete
  permission manager read, write
  permission user    read

  field id        : int    readonly
  field firstname : string filterable {
    example like "Anna" "firstname contains Anna"
  }
  field lastname  : string filterable
  field age       : int    filterable {
    example gt 50 "older than 50"
  }
  field city      : string options=cities filterable {
    example eq "Berlin" "exact city match"
  }
  field role      : string options=roles

  relation notes : has_many note bind=id -> person_id

  meta roles  from role  key label order_by label
  meta cities from city  name name order_by name

  ai "scope" "Employees, customers and contacts."

  aliases [ "mitarbeiter", "kunden" ]
}
```

---

## View DSL

A view file describes how a domain is presented — toolbar actions,
tables, forms and navigation. One domain typically has two views: a
list view and a detail view.

### Basic structure

```
view <name> "<title>" over <domain> {
  auto_refresh on <domain>.changed

  toolbar { ... }

  table -> <domain>.rows { ... }
  section "<title>" { ... }
  row { ... }
  widget <kind> bind=<field> [mods...]
}
```

### Toolbar

```
toolbar {
  new     "New"    -> person_detail as tab
  refresh "Reload"
  save    "Save"
  delete  "Delete"
}
```

### Table

```
table -> person.rows {
  on_select -> person_detail bind=person.id as tab

  column person.id        "ID"        width=60
  column person.firstname "Firstname" width=140
  column person.age       "Age"       width=70  format=number:0
  column person.net_worth "Worth"     width=110 format=currency
}
```

Column formats: `number:0`, `number:2`, `currency`, `date`, `datetime`, `bool`.

### Detail layout

```
section "Contact" {
  row {
    widget text   bind=person.email
    widget text   bind=person.phone
  }
  row {
    widget select bind=person.city
  }
}
```

Widget kinds: `text`, `textarea`, `select`, `checkbox`, `number`, `date`.

### Full list-view example

```
view person_list "Persons" over person {

  auto_refresh on person.changed

  toolbar {
    new     "New"    -> person_detail as tab
    refresh "Reload"
  }

  table -> person.rows {
    on_select -> person_detail bind=person.id as tab

    column person.id        "ID"       width=60
    column person.firstname "Firstname" width=140
    column person.lastname  "Lastname"  width=140
    column person.email     "Email"     width=220
    column person.city      "City"      width=120
    column person.age       "Age"       width=70  format=number:0
  }
}
```
