---
layout: default
title: Components
nav_order: 3
---

# Components

## oosgql — GraphQL gateway

**Location:** `apps/oosgql/` · **Port:** 4000 · **Runtime:** Bun + Hono

oosgql turns domain definitions into a live GraphQL API. On startup it
loads every domain from `oos.domain`, builds a GraphQL schema via
`packages/oos-gql-ts`, and starts serving. When `oos.domain.changed`
arrives on NATS, it rebuilds the schema in place.

**NATS subjects handled:**

| Subject | Description |
|---|---|
| `oos.cmd.gql.query` | Execute a GraphQL query |
| `oos.cmd.gql.mutation` | Execute a GraphQL mutation |

**HTTP endpoints (internal):**

| Endpoint | Description |
|---|---|
| `GET /health` | `{status: "ok"}` |

oosgql never speaks directly to any client. All traffic arrives via
oosai which acts as a proxy for the `oos.cmd.gql.*` subjects.

---

## oosai — Embedding and command hub

**Location:** `apps/oosai/` · **Port:** 4100 · **Runtime:** Bun + Hono

oosai is the central nervous system. It handles every NATS subject the
clients send, maintains the vector store, and drives the event
embedding pipeline.

### Boot sequence

1. Load config (`OOSAI_*` env vars, sane defaults for development).
2. Connect to PostgreSQL and Ollama.
3. Run backfill — re-embed all domains, views and global prompts.
4. Start notify listeners — react to domain/view/prompt changes.
5. Start event listeners — one NATS subject per active event mapping.
6. Start the Hono HTTP server (internal health only).
7. Start the NATS command handler.

### NATS subjects handled

**Domain / View / Prompt CRUD:**

| Subject | Description |
|---|---|
| `oos.cmd.domain.list` | List all domains |
| `oos.cmd.domain.load` | Load one domain by name |
| `oos.cmd.domain.save` | Save a domain (triggers re-embed + notify) |
| `oos.cmd.domain.delete` | Delete a domain |
| `oos.cmd.view.list` | List all views |
| `oos.cmd.view.load` | Load one view |
| `oos.cmd.view.save` | Save a view |
| `oos.cmd.view.delete` | Delete a view |
| `oos.cmd.prompt.list` | List global prompts |
| `oos.cmd.prompt.save` | Save a global prompt |
| `oos.cmd.prompt.delete` | Delete a global prompt |

**RAG / Index:**

| Subject | Description |
|---|---|
| `oos.cmd.search` | Cosine search over embedded domain chunks |
| `oos.cmd.global` | Return all global prompt chunks |
| `oos.cmd.domains` | Return the full domain index (name + description) |
| `oos.cmd.views` | Return the full view index |

**Event system:**

| Subject | Description |
|---|---|
| `oos.cmd.event_type_grammar.list` | List all event type grammars |
| `oos.cmd.event_type_grammar.load` | Load one grammar by name |
| `oos.cmd.event_type_grammar.save` | Save a grammar |
| `oos.cmd.event_type_grammar.delete` | Delete a grammar |
| `oos.cmd.event_streams.list` | List streams, optionally by mapping |
| `oos.cmd.event_streams.create` | Create a stream |
| `oos.cmd.event_streams.delete` | Delete a stream |
| `oos.cmd.event_mappings.list` | List event mappings |
| `oos.cmd.event_mappings.set_types` | Set accepted event types for a mapping |
| `oos.cmd.event.insert` | Insert an event and trigger embedding |
| `oos.cmd.event.search` | Cosine search over embedded events |
| `oos.cmd.event.schemas` | Return tag-filtered event type grammars |
| `oos.cmd.event.stream_events` | Return event history for a stream |

**GraphQL proxy:**

| Subject | Description |
|---|---|
| `oos.cmd.gql.query` | Proxy a GraphQL query to oosgql |
| `oos.cmd.gql.mutation` | Proxy a GraphQL mutation to oosgql |

**Error convention:** every reply is `{ ok: boolean, error?: string, ...data }`.
Handlers never throw into the NATS layer.

---

## oosd — Designer

**Location:** `apps/oosd/` · **Runtime:** Electrobun + Bun + React 19 + Mantine 9

oosd is the authoring and administration tool. It is an Electrobun
desktop app with a three-process model: a native shell, a Bun HTTP
gateway and a React UI.

**Navigation panels:**

| Panel | Purpose |
|---|---|
| Domain | Edit `.domain` DSL files with Monaco and live diagnostics |
| View | Edit `.view` DSL files |
| Event Types | Grammar library — language rules for event classification |
| Mapping Types | Which event types each mapping accepts |
| Settings | NATS URL, LLM endpoint, model picker |
| Demo | Install the internal schema and demo dataset |

**Settings (stored as `oosd-settings-v2`):**

```ts
{
  natsUrl: string   // default: nats://localhost:4222
}
```

LLM settings (base URL, API key, model) are also stored here, used by
the oosd chat agent.

**Chat window:** oosd opens a detached Electrobun `BrowserWindow` for
the AI chat. The agent inside oosd understands the DSL and can list
domains, load sources, save views and generate DSL code blocks that
open directly in the editor.

---

## oos — Chat client

**Location:** `apps/oos/` · **Runtime:** Electrobun + Bun + React 19 + Mantine 9

oos is the end-user application. Users ask questions in natural
language; the built-in ReAct agent loop resolves the intent into
schema lookups and GraphQL calls and renders the result as an
interactive board.

**Settings (stored locally):**

```ts
{
  natsUrl:    string  // default: nats://localhost:4222
  llmBaseUrl: string
  llmApiKey:  string
  llmModel:   string
}
```

**Agent tools:**

| Tool | Description |
|---|---|
| `oos_schema_search` | Cosine search over domain chunks via NATS |
| `oos_query` | Execute a GraphQL query via NATS |

See [The AI assistant](./the-ai-assistant.html) for the full agent design.

---

## Shared packages

| Package | Purpose |
|---|---|
| `packages/oos-dsls-ts` | Langium grammar, AST, `parseDomain`, `parseView`, `renderLLMChunk` |
| `packages/oos-gql-ts` | `buildSchema`, query/mutation resolvers, permission checks |
| `packages/oos-embed-ts` | OpenAI-compatible embed client, pgvector helpers |
| `packages/oos-ui-ts` | Shared React UI primitives |
