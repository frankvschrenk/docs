---
layout: default
title: The toolbox
nav_order: 10
---

# The toolbox

OOS ships a dedicated authoring tool alongside the main client and
server. It is not part of the runtime — it exists to help you write
and iterate on context and screen definitions without ever touching
the database directly.

## ooso — the synthetist

**ooso** is the authoring tool for OOS definitions. Think of it as the
IDE for context and screen files: live preview, validation against the
XSD, and batch imports for CI pipelines.

### GUI mode

Start it without arguments:

```bash
ooso
```

Three panels open:

- **CTX** — pick a `groups.xml`, import context definitions.
- **DSL** — open a screen file, feed it sample data, watch the
  rendered Fyne UI update as you edit.
- **Theme** — edit the UI theme, switch between dark and light,
  save the result back into `oos.ctx`.

The live preview is the fastest loop for authoring screens. You edit
the XML, the preview re-renders, you see what changed. No build, no
restart, no client navigation.

### CLI mode

The same operations are scriptable. Useful in CI or when you have a
lot of files to import:

```bash
ooso add group oos-admin
ooso add context person
ooso dsl add person-detail.dsl.xml
ooso dsl add-dir ./dsl
ooso dsl preview --dsl some.dsl.xml --data sample.json
```

`dsl add-dir` walks a directory and imports every `*.dsl.xml` it
finds. This is the pattern we use to keep the demo in sync.

### When you use ooso

- Authoring new context files.
- Iterating on a screen layout.
- Switching or tweaking the UI theme.
- Validating that a set of context files parses and builds a valid
  GraphQL schema before pushing them into production.

### When you don't

You don't need ooso to run OOS. Once definitions are in the database,
the client and the backend do their thing. ooso is for the design
phase.

## What's next

- [Running OOS](./running-oos.html) — setup for the core system.
- [Writing contexts](./writing-contexts.html) and
  [Designing screens](./designing-screens.html) — what ooso helps you
  produce.
