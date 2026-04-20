---
layout: default
title: How it works
nav_order: 4
---

# How it works

OOS has three long-running pieces and one authoring tool. Everything
is written in Go. The glue between them is a small number of
well-defined wire formats: GraphQL, REST+JSON, XML for definitions,
and a vector store for the AI's schema retrieval.

## The long-running pieces

**oos** is the desktop client. It owns the UI, the chat with the AI
assistant, and the user's session. It is a Fyne application — native
windows, native menus, GPU-rendered via Skia. It talks to oosp over
HTTPS, authenticates against an OIDC provider (we develop against
Zitadel), and never touches a database directly.

**oosp** is the backend. It owns the GraphQL schema, the context and
screen definitions, the permission matrix, and the connection to the
data sources. It serves the AI assistant's tools (query, save,
schema_search, …) over REST. It keeps a vector store of schema chunks
so the assistant can retrieve the right context for any
natural-language request.

**PostgreSQL** is where everything lives. Data tables — the domain
records your application is about. Meta tables — the reference lists
behind dropdowns. And three OOS-internal tables: `oos.ctx` (context
definitions as XML), `oos.dsl` (screen definitions as XML), and
`oos.oos_schema` (the AI-facing schema chunks, with their embeddings).

A **vector store** holds the embeddings of the schema chunks. OOS uses
PostgreSQL with the `pgvector` extension by default; any compatible
vector backend works.

## The authoring tool

**ooso** is the synthetist — the authoring tool for context and screen
definitions. It has a GUI for live editing and a CLI for batch imports.
You use it when you are designing the system, not when you are running
it.

## The two data-flow pipelines

Two things happen every time a context definition changes. Both start
from the same row in `oos.ctx` and fan out:

**Pipeline 1: GraphQL.** The context XML is parsed into an AST. From
the AST, oosp builds the GraphQL schema. Queries, filters, inserts,
updates, deletes — all generated, all reflecting exactly what your
definition says. When you save a changed context, the schema rebuilds
itself without a server restart.

**Pipeline 2: AI-facing schema chunks.** The same AST is also rendered
as plain-text chunks: one per context, describing its fields, its
filter shape, its dropdown metadata, a full example query. The chunks
are embedded using a granite embedding model and stored with their
vectors in `oos.oos_schema`. When the assistant is asked something,
it first finds the most relevant chunks by vector similarity, then
uses them to construct a grounded query.

## A typical request

A user opens the chat and types *"zeig mir alle Berliner Kunden über
50"*. What happens:

1. The client sends the message to the AI assistant's session, which
   already carries the user identity, the role, the permission matrix,
   the global prompts, and (depending on strategy) the full schema or
   a retrieval hint.
2. The assistant picks the `person_list` context — either from the
   embedded schema or by calling `oos_schema_search` for it.
3. It builds a GraphQL query with the suffix-argument filters from
   the schema: `{ person_list(city_like: "Berlin", age_gt: 50) { ... } }`.
4. It calls the `oos_query` tool. oosp runs the query against the
   configured data source and returns the rows.
5. The board opens in the client with the result. The assistant
   doesn't narrate — the data is the answer.

A mutation request ("ändere die Rolle von Person 5 auf Manager") takes
a different path: the assistant proposes a preview via
`oos_ui_change_required`, the user confirms, and only then does
`oos_ui_save` reach the backend. The mutation itself is built by
oosp, not by the LLM.

## Where permissions enter

Every context declares what each role may do:

```xml
<permission role="admin"   actions="read,write,delete"/>
<permission role="manager" actions="read,write"/>
<permission role="user"    actions="read"/>
```

The client reads this table into the AI's system prompt so the
assistant knows ahead of time what the user is allowed to do — and
can say so. The server reads the same table on every `/save` and
`/mutation` request and rejects anything outside the role's rights.
If the client is out of sync, misleading or compromised, the server
still holds the line.

## What's next

- [Writing contexts](./writing-contexts.html) — the author's view.
- [Designing screens](./designing-screens.html) — the designer's view.
- [The AI assistant](./the-ai-assistant.html) — how the LLM is wired in.
