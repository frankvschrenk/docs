---
layout: default
title: Glossary
nav_order: 11
---

# Glossary

The vocabulary you'll meet throughout the OOS documentation and code.

### AST (Abstract Syntax Tree)
The parsed, in-memory form of a set of context files. The GraphQL
schema, the permission matrix and the AI-facing schema chunks are
all derived from the AST.

### chunk (schema chunk)
A plain-text description of one context, rendered from the AST for
the AI assistant to read. Every chunk contains the context name,
the query shape, the filter examples, meta queries, and permissions.
Chunks live in `oos.oos_schema` along with their embeddings.

### collection
A context whose `kind="collection"`. It represents a list of records —
a table view, essentially. By convention, collection names end in
`_list`.

### context
The core unit of OOS. A context is an addressable view onto your data,
either a collection (list) or an entity (single record). Contexts are
declared in XML, live in `oos.ctx`, and drive the GraphQL schema, the
screen rendering and the AI's knowledge of what exists.

### CTX
Shorthand for a context definition file — the XML document stored in
`oos.ctx`. Sometimes the word is also used for the DB table itself.

### DSL
Short for Domain-Specific Language. OOS has two of them:
- **Context DSL** — the XML for describing entities, fields,
  permissions and relations. See [writing contexts](./writing-contexts.html).
- **Screen DSL** — the XML for describing UI layouts, forms,
  tables and widgets. See [designing screens](./designing-screens.html).

### eino
The Go agent framework we use inside the client for the ReAct-style
AI assistant. Provides the tool-calling loop, the message history
and the model interface.

### entity
A context whose `kind="entity"`. It represents a single record with
all its fields visible for viewing and editing. By convention, entity
names end in `_detail`.

### envelope
The JSON shape oosp returns when the client asks for a screen to
render: a `content` section with the record data, a `meta` section
with dropdown options, and a `dsl` section with the screen layout.
The client loads all three into the state at once.

### Fyne
The Go GUI toolkit the desktop client is built on. Native windows,
native input, rendered through Skia on the GPU. Not a webview.

### GraphQL
How the client talks to oosp about data. The schema is generated from
the context AST at runtime, which means it always reflects the current
definitions — add a field to a context and it appears in the schema
on the next save.

### granite embedding
The embedding model we use to turn schema chunks into vectors for
retrieval. IBM's granite-embedding line; small, fast, good enough for
the chunk-matching problem. Runs wherever your OpenAI-compatible
endpoint runs.

### IAM
Identity and access management. OOS authenticates through OIDC, uses
PKCE, and assumes a separate identity provider — we develop against
Zitadel.

### meta / meta table
A reference list used to populate dropdowns. Declared inside an
entity context with `<meta name="roles" table="role" .../>`. At
runtime oosp exposes a `meta_<n>` GraphQL query that returns
`{value, label}` pairs.

### oos
The desktop client. Fyne-based. Owns the UI, the chat with the
assistant, and the user's session.

### oosp
The backend. Serves GraphQL, manages the context and screen
definitions, keeps the schema chunks up to date, exposes the AI
tools as REST endpoints.

### ooso
The synthetist. The authoring tool for context and screen
definitions, with GUI and CLI modes.

### pgvector
The PostgreSQL extension OOS uses by default for vector similarity
search over schema chunks. Any compatible vector backend also works,
but pgvector keeps the stack to a single database.

### PKCE (Proof Key for Code Exchange)
The OAuth flow OOS uses for authentication. Appropriate for desktop
apps because it doesn't require a client secret on the device.

### RAD (Rapid Application Development)
The category OOS belongs to: tools that let you build full
applications from high-level descriptions, with minimal hand-written
code. OOS's twist is adding AI to the data-access layer.

### ReAct
A pattern for AI agents where the model alternates between thinking
and acting (calling tools). OOS's assistant is a ReAct agent built
on eino.

### source
A context attribute naming the underlying database table. Multiple
contexts may share the same source — the demo has `person_list` and
`person_detail` both with `source="person"`.

### Zitadel
The OIDC provider we develop against. OOS works with any standard
OIDC provider; Zitadel is just what ships in our dev environment.
