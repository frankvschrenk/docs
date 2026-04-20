---
layout: default
title: The AI assistant
nav_order: 7
---

# The AI assistant

OOS has an AI assistant baked in. It is not a chatbot with a shallow
connection to the database — it is a grounded agent that talks to
oosp through a narrow set of tools and obeys your context and permission
definitions. This page is what it does, how it does it, and what to
know when you author prompts.

## What the assistant is good at

The assistant earns its keep on three kinds of work:

1. **Natural language to query.** "Show me all Berlin customers over
   50" becomes a GraphQL query with the right filters and the right
   context. The user never types GraphQL.
2. **Multi-step lookups.** "For every note Anna left in the last
   quarter, what's the title?" becomes two or three tool calls with
   intermediate reasoning. No one writes the join logic.
3. **Preflight feedback.** "Delete this record" becomes "your role
   can't delete — you'd need manager or admin". The permission
   matrix is in the prompt, the assistant reads it, the user finds
   out before anything fails at the server.

## What the assistant does *not* do

- It does not build mutations from scratch. Changes to data always go
  through `oos_ui_change_required` (preview) and `oos_ui_save`. The
  actual GraphQL mutation is built by oosp from the preview payload,
  not by the LLM. This keeps the LLM from inventing fields.
- It does not touch the database directly. All data access is through
  tools; all tools go through oosp.
- It does not narrate data. When a query returns, the board renders
  the data and the assistant stays quiet. Long summaries of rows are
  noise.

## The tools it has

Defined in `oos/aiassist/tools.go` and stable as of writing:

- `oos_schema_search` — find the schema chunks relevant to an
  intent. Always called first in RAG mode; in compact/full mode the
  schema is already in the prompt.
- `oos_query` — run a GraphQL query and open the result in the board.
- `oos_ui_change_required` — preview a change in the current board
  without writing.
- `oos_ui_save` — persist the previewed change after user
  confirmation.
- `oos_new` — open an empty detail screen for a context.
- `oos_delete` — delete a record after explicit user confirmation.
- `oos_render` — render arbitrary JSON in the board (used for
  assistant-generated views).
- `oos_stream_append` / `oos_vector_search` — the event system
  hooks for workflows that log over time.
- `oos_system_status` — a diagnostic helper.

## How the prompt is assembled

At session start the client composes the system prompt from three
sources:

1. **The user section** — who is talking, which role they hold, and
   the resolved permission matrix for every context. This is
   generated from the AST, not hardcoded. Authored in
   `oos/tools/schema.go`.
2. **The schema block** — in the default `compact` strategy, every
   context contributes its key lines (GraphQL query shape, filter
   examples, context header). In `full` strategy the whole chunk
   set goes in. In `rag` strategy only a hint is added and the
   assistant retrieves as needed.
3. **The instructions** — rendered from the `<prompt>` entries in
   `global.conf.xml`. These are authored content, not code.

The order matters: schema before instructions, because instructions
reference schema vocabulary.

## The global prompts

`global.conf.xml` is the admin-editable behaviour file. It lives in
`oos.ctx` and reloads whenever you change it. The prompts that ship
in the demo:

- `identity` — who the assistant is and its grounding principle.
- `schema_discovery` — when to call schema search versus rely on
  what's already in the prompt.
- `query_behavior` — collection vs entity, which fields to request.
- `filter_syntax` — suffix arguments only, never `where` blocks.
- `dropdowns_and_meta` — include every meta_NAME sub-query for
  dropdown fields.
- `mutation_behavior` — the four-step write workflow.
- `deletion` — irreversible, needs explicit user intent.
- `permissions` — consult the user's role before proposing writes.
- `response_style` — short replies, Markdown only, no LaTeX.
- `language` — reply in the user's language.

Changing any of these changes the assistant's behaviour immediately.
That is the point: behaviour is config, not code.

## How the schema chunks are built

For every context in `oos.ctx`, oosp renders a chunk of plain text
and stores it in `oos.oos_schema` along with an embedding vector.
The chunk contains everything the assistant needs to write a valid
query:

- Context name, kind, source and aliases.
- List of allowed query fields.
- A full example `{ context_name { fields } }` query.
- Filterable fields with per-field examples.
- Meta queries for dropdowns, with a combined example showing how
  to fetch a detail record and all its meta tables in a single
  GraphQL request.
- The resolved permissions line.

The chunks regenerate automatically whenever a context changes,
because oosp listens to PostgreSQL notifications on `oos.ctx`.

## Choosing a schema strategy

The `LLMSchemaStrategy` setting controls how much schema goes into
the prompt at session start:

- **compact** — default. Every context contributes ~3-4 lines. Scales
  to hundreds of contexts.
- **full** — the complete chunk set, one big block. Good for
  powerful models with generous context windows. Use when you know
  the total fits.
- **rag** — no schema at all, just a hint to call
  `oos_schema_search`. Minimum token usage, one extra tool call per
  request.

For small models like Gemma 2, compact is the sweet spot — the
schema is already there, and the assistant doesn't waste a turn
retrieving it.

## Why the LLM never sees the data

Sensitive data only moves between oos and oosp. Queries are run by
oosp against the configured data source, and the results come back
over REST to the client. The LLM sees the user's message, the
schema, the tool results — but the data itself is never sent to any
remote model endpoint. This matters especially when you run the
assistant against a hosted LLM: no row of your database ends up in
someone else's logs.

## What's next

- [Security and roles](./security-and-roles.html) — how the role
  information in the prompt gets there and what the server does with
  it independently.
- [The toolbox](./the-toolbox.html) — the ooso authoring tool for
  contexts and screens.
