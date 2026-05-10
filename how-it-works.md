---
layout: default
title: How it works
nav_order: 2
---

# How it works

OOS has four long-running services and one authoring tool. The glue
between them is **NATS** — all inter-service communication goes through
NATS Request-Reply subjects. No service knows the address of another;
every client only needs a NATS URL.

## The services

**oosgql** is the GraphQL gateway (port 4000). It reads domain
definitions from the database, builds a live GraphQL schema, and
handles queries and mutations. When a domain changes, a NATS
notification triggers an instant schema rebuild — no restart needed.

**oosai** is the embedding and command hub (port 4100). It embeds
domain and view definitions into pgvector for semantic retrieval,
listens for incoming events and embeds those too, and exposes the full
CRUD surface of OOS's internal tables as NATS subjects. Every NATS
command the clients send — whether it is loading a domain, inserting an
event or running a GraphQL proxy — is handled here.

**oos** is the chat client (Electrobun + React). End users type natural
language; a local ReAct agent loop turns that into schema lookups and
GraphQL calls. The result is rendered as an interactive board.

**oosd** is the designer (Electrobun + React). Developers write
`.domain` and `.view` files, install the demo dataset, manage event
type grammars and inspect event streams. It also hosts a chat window
backed by its own ReAct agent that understands the DSL.

## The communication topology

```
oos  ──NATS──▶ oosai  ──DB──▶  PostgreSQL + pgvector
oosd ──NATS──▶ oosai
               oosai  ──HTTP──▶ oosgql  (GQL proxy)
               oosai  ──Embed──▶ Ollama (embedding model)
oosgql  ──DB──▶  PostgreSQL
```

Neither `oos` nor `oosd` has a database URL or a direct HTTP connection
to any backend. They only need `nats://localhost:4222`.

## A typical query

A user opens the oos chat and types _"show me all customers from
Berlin"_. What happens:

1. The agent calls `oos_schema_search` — oosai does a cosine search
   over the embedded domain chunks and returns the `customer` context.
2. The agent builds a GraphQL query using the field and filter
   information in that chunk and calls `oos_query`.
3. oosai forwards the query to oosgql via `oos.cmd.gql.query`.
4. oosgql runs the query against PostgreSQL and returns the rows.
5. oos renders the result as a board.

## A typical domain change

A developer edits `customer.domain` in oosd and saves it. What happens:

1. oosd sends `oos.cmd.domain.save` via NATS to oosai.
2. oosai writes the new source to `oos.domain` in the database.
3. oosai publishes `oos.domain.changed`.
4. oosai's notify listener picks up the change and re-embeds the domain chunk.
5. oosgql's notify listener picks up the change and rebuilds the GraphQL schema.

The running oos client picks up the fresh schema on its next query.

## The database

OOS uses PostgreSQL with the `pgvector` and `hstore` extensions.

Internal tables live in the `oos` schema:

| Table | Purpose |
|---|---|
| `oos.domain` | Domain DSL sources |
| `oos.view` | View DSL sources |
| `oos.global_prompt` | Standing instructions injected into every LLM system prompt |
| `oos.event_type_grammar` | Grammar rules for event classification |
| `oos.event_mappings` | Which event sources are active and which event types they accept |
| `oos.event_streams` | Named event streams with context tags |
| `oos.oos_domain_schema` | Embedded domain chunks (pgvector) |
| `oos.oos_global_schema` | Embedded global prompt chunks (pgvector) |

Application data lives in the `public` schema — whatever tables your
domain definitions point to.
