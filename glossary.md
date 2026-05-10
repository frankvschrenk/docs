---
layout: default
title: Glossary
nav_order: 9
---

# Glossary

**agent** — The ReAct loop running inside oos or oosd that turns
natural language into tool calls (schema search, GraphQL queries,
DSL saves) and assembles the result into a response.

**backfill** — The process oosai runs on startup to embed any domain,
view, global prompt or event records that are not yet represented in
the vector store.

**domain** — A named description of one data entity. Written in the
domain DSL (`.domain`). Describes the database table, fields, types,
permissions, relations, meta sources and AI hints. The source of
truth for everything OOS generates: GraphQL schema, LLM chunks, UI.

**domain chunk** — The plain-text representation of a domain, produced
by `renderLLMChunk` in `packages/oos-dsls-ts`. Embedded as a vector
and used for semantic retrieval by the agent.

**event** — One row in an event source table. Events are inserted via
`oos.cmd.event.insert`, embedded by oosai, and searchable by
semantic similarity.

**event mapping** — A named configuration (`oos.event_mappings`) that
connects a source table to the event system and lists which event
type grammars it accepts.

**event stream** — A named context within a mapping, typically a case
file, session or thread. Identified by a stream ID
(e.g. `fall-2024-0042`).

**event type grammar** — A DSL source in `oos.event_type_grammar` that
describes one class of event: required/optional fields, context tags
and examples. Edited in oosd's Event Types panel.

**global prompt** — A row in `oos.global_prompt`. Every global prompt
is injected verbatim into the LLM system prompt on every turn.
Used for standing instructions: deletion policy, filter conventions,
response style, permission reminders.

**knowledge sandwich** — The system prompt design used in oos. Global
standing instructions and the domain index form the stable outer
layers; on-demand schema chunks retrieved by `oos_schema_search`
fill the middle as the agent works.

**meta source** — A lookup table backing a dropdown field. Declared in
the domain with the `meta` keyword. Loaded by oosgql to populate
dropdown options.

**NATS** — The message bus used for all inter-service communication.
OOS uses NATS Request-Reply exclusively. All clients only need a
NATS URL; they do not know the addresses of individual services.

**oosai** — The embedding and command hub. Handles all NATS subjects,
maintains the vector store, drives the event embedding pipeline, and
proxies GraphQL calls to oosgql.

**oosd** — The designer desktop app. Used by developers to write domain
and view DSL files, manage event type grammars, and install the demo
dataset.

**oosgql** — The GraphQL gateway. Builds a live GraphQL schema from
domain definitions and executes queries and mutations against
PostgreSQL.

**oos** — The chat desktop app. End users query and edit data in
natural language through a local ReAct agent.

**pgvector** — A PostgreSQL extension for storing and querying
embedding vectors. Used by oosai for domain chunks, global prompt
chunks and event embeddings.

**ReAct** — Reasoning + Acting. The agent loop pattern: ask LLM,
if it requests tools run them, append results, repeat. Used in both
oos and oosd.

**renderLLMChunk** — The function in `packages/oos-dsls-ts` that
converts a `DomainDef` into a structured plain-text block for
embedding. Output is deterministic and versioned.

**view** — A named description of how a domain is presented.
Written in the view DSL (`.view`). Describes toolbar actions, table
columns, form sections and widget bindings.
