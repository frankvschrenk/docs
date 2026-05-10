---
layout: default
title: Home
nav_order: 1
---

# onisin OS Documentation

onisin OS (OOS) is an AI-first enterprise data system. You describe
your data model in small declarative DSL files. From those files, OOS
generates a live GraphQL schema, renders a desktop UI, and gives an AI
assistant everything it needs to answer questions and edit records in
natural language — all without touching a database directly.

## The mental model

A **domain** is the source of truth for one entity — say, `person`.
It describes fields, types, permissions, relations and AI hints.
A **view** describes how that domain is displayed: toolbar buttons,
column layout, form widgets. Everything is derived from these two
files.

When a domain changes, the whole stack reacts automatically:

- **oosgql** rebuilds the GraphQL schema.
- **oosai** re-embeds the domain chunk for AI retrieval.
- The running UI picks up the change without a restart.

## Where to go from here

- [How it works](./how-it-works.html) — architecture and data flow.
- [Components](./components.html) — each service explained.
- [The DSL](./the-dsl.html) — writing `.domain` and `.view` files.
- [The AI assistant](./the-ai-assistant.html) — how the LLM is wired in.
- [The event system](./event-system.html) — event types, streams and mappings.
- [Running OOS](./running-oos.html) — setup and startup order.
- [Security and roles](./security-and-roles.html) — permissions.
- [Glossary](./glossary.html) — terms used throughout the docs.
- [License](./license.html) — BSL 1.1.
