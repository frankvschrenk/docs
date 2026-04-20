---
layout: default
title: Running OOS
nav_order: 9
---

# Running OOS

OOS is a native app plus a backend. This page walks through getting
a working setup from a fresh clone.

## Prerequisites

- **Go** 1.22 or newer.
- **PostgreSQL** 14 or newer, with the **pgvector** extension.
  Used for OOS-internal state (`oos.*` schemas), the schema-chunk
  embeddings, and optionally your application data.
- An **LLM endpoint** that speaks OpenAI-compatible chat completions.
  For development we use Ollama with Gemma; for deployment, any
  OpenAI-compatible provider works.
- An **OIDC provider**. We develop against Zitadel. Keycloak works
  the same way; hosted providers (Auth0, Okta) also work as long as
  they speak standard OIDC with PKCE.
- macOS, Linux or Windows for the desktop client.

Supported data sources today: **PostgreSQL**, **Oracle** and
**MariaDB** — plus any other backend you wire up behind the same
datasource interface.

## The repository layout

```
onisin/
├── oos/         desktop client (Fyne)
├── oosp/        backend (REST + GraphQL)
├── oos-common/  shared: AST, GraphQL builder, plugin transport
├── oos-dsl/     DSL runtime for screens
├── oos-dsl-base/ DSL parser and nodes
├── oos-run/     installer and seed
├── ooso/        synthetist (CTX/DSL authoring)
├── oos.xsd      schema for context files
└── setup.toml   runtime configuration
```

## Configuration — setup.toml

Every binary reads the same `setup.toml`. The interesting sections:

```toml
[oosp]
addr = "127.0.0.1:9100"
dsn  = "postgres://oosp:...@localhost/onisin"

[oosp.datasources.demo]
type     = "postgres"
host     = "localhost"
database = "demo"
user     = "demo"
password = "..."

[oosp.vector]
backend = "pgvector"
dsn     = "postgres://oosp:...@localhost/onisin"

[oosp.llm]
url         = "http://localhost:11434"
embed_model = "granite-embedding"

[oos]
oosp_url = "http://localhost:9100"

[iam]
issuer_url = "https://your-zitadel/oauth/v2"
client_id  = "..."
scope      = "openid profile email groups"
```

## First-time setup

Start Postgres (with pgvector enabled). Then:

```bash
make build               # builds oos, oosp, ooso
./dist/oosp              # start the backend
./dist/oos-run_macos --install --zip staging/macos.zip  # install the client
```

The installer puts `oos` and the supporting tools into
`~/.local/bin` (or the platform equivalent). It's the path that lets
you run `oos` from anywhere.

## Seeding

A fresh database is empty. To populate it with the demo (a `person`
context and a `note` context, plus the permissions, screens, global
prompts):

```bash
./dist/oosp --seed
```

The seeder writes into `oos.ctx` and `oos.dsl`. oosp listens for
changes and regenerates the GraphQL schema and the AI-facing chunks
automatically — no restart needed.

If you already have data in `oos.oos_schema` and want a clean
rebuild:

```sql
TRUNCATE oos.oos_schema;
```

Then restart oosp and the backfill regenerates every chunk.

## Running the client

```bash
oos
```

The client opens a login window, redirects you to your OIDC provider,
receives the token, and establishes a session. You'll see the
dashboard — activity panel on the right, board area in the middle,
AI chat in its own tab.

## Tearing down

Because the system state is almost entirely in Postgres, cleanup is
a database operation:

```sql
DROP SCHEMA oos CASCADE;
```

followed by a seed cycle to put the defaults back.

## What about credentials for data sources?

If you configure Vault (`[iam.vault]`), OOS can fetch per-session
certificates using the user's JWT for mutual TLS against your
databases. This means the user's identity travels with every SQL
statement, not just the session-wide shared credential. The dev
environment does not require Vault; `[unsecure_mode] = true` skips
the whole dance.

## Common gotchas

- **Client can't reach oosp.** Check `[oos] oosp_url` and that oosp
  is actually listening. `curl http://localhost:9100/health` should
  return `{"status":"healthy"}`.
- **No groups in JWT.** The demo falls back to username mapping:
  `admin` → `oos-admin`, `user` → `oos-user`. For production,
  configure the groups claim in your IAM.
- **Dropdowns empty.** The meta tables (role, department, city,
  etc.) are expected to exist in your demo database with the
  columns declared in the `<meta>` elements of the context file.

## What's next

- [The toolbox](./the-toolbox.html) — the ooso authoring tool.
- [Security and roles](./security-and-roles.html) — the IAM details.
