---
layout: default
title: Running OOS
nav_order: 7
---

# Running OOS

## Prerequisites

- **Bun** ≥ 1.2
- **PostgreSQL** ≥ 14 with the `pgvector` and `hstore` extensions
- **NATS Server** — `brew install nats-server` on macOS
- **Ollama** with a tools-capable chat model (e.g. `llama3.1`,
  `qwen2.5`) and the `granite-embedding:latest` embedding model
- An OpenAI-compatible LLM endpoint (Ollama is fine for local work)

## Repository layout

```
onisin/
├── apps/
│   ├── oos/       — chat client (Electrobun)
│   ├── oosd/      — designer (Electrobun)
│   ├── oosgql/    — GraphQL gateway (Bun + Hono, port 4000)
│   └── oosai/     — embedding + command hub (Bun + Hono, port 4100)
├── packages/
│   ├── oos-dsls-ts/   — Langium grammar, parsers, LLM chunk renderer
│   ├── oos-gql-ts/    — GraphQL schema builder, resolvers, permissions
│   ├── oos-embed-ts/  — embedding client, pgvector helpers
│   └── oos-ui-ts/     — shared React UI primitives
├── oosfs/         — Go MCP server for repo tooling
├── demo.toml      — local DB config (used by oosfs MCP tools)
└── package.json   — Bun workspace root
```

## Install dependencies

```bash
bun install
```

Run this once at the repo root. Bun installs all workspace packages
in one pass.

## Configuration

Each service reads environment variables with sane defaults for local
development. For production, set them explicitly.

**oosgql** (`apps/oosgql/src/config.ts`):

| Variable | Default | Description |
|---|---|---|
| `OOSGQL_PORT` | `4000` | HTTP port |
| `OOSGQL_HOST` | `localhost` | Listen address (`0.0.0.0` required in Kubernetes) |
| `OOSGQL_PG_URL` | `postgres://postgres:demo@localhost:5432/onisin` | Database URL |
| `OOSGQL_NATS_URL` | `nats://localhost:4222` | NATS URL |

**oosai** (`apps/oosai/src/config.ts`):

| Variable | Default | Description |
|---|---|---|
| `OOSAI_PORT` | `4100` | HTTP port |
| `OOSAI_HOST` | `localhost` | Listen address (`0.0.0.0` required in Kubernetes) |
| `OOSAI_PG_URL` | `postgres://postgres:demo@localhost:5432/onisin` | Database URL |
| `OOSAI_NATS_URL` | `nats://localhost:4222` | NATS URL |
| `OOSAI_EMBED_URL` | `http://localhost:11434/v1` | Ollama base URL |
| `OOSAI_EMBED_MODEL` | `granite-embedding:latest` | Embedding model |
| `OOSAI_EMBED_KEY` | _(empty)_ | API key — leave empty for local Ollama |

## Startup order

Services are independent but the event system's backfill requires the
database schema to exist before oosai starts. Safe order:

```bash
# 1. Infrastructure
nats-server

# 2. Backend services (separate terminals)
cd apps/oosgql && bun run dev
cd apps/oosai  && bun run dev

# 3. Apps
cd apps/oosd && bun run dev   # designer
cd apps/oos  && bun run dev   # chat client
```

`oosgql` and `oosai` can start in any order relative to each other.

## Installing the demo

The demo dataset provides two domains (`person`, `note`), reference
tables, 10 persons, 12 notes, and police and support event data. It
is installed from inside oosd:

1. Open oosd.
2. Go to **Settings** → set the NATS URL if needed (`nats://localhost:4222`).
3. Go to **Demo**.
4. Click **Install internal schema** — writes all `oos.*` tables.
5. Click **Install demo tables and data** — writes the `public` schema,
   reference tables, persons, notes, events, and all DSL sources.

Both steps are idempotent. Re-running resets the demo data.

**Important:** oosai must be running before you install the demo.
The installer triggers event embedding; if oosai is not listening,
the events are inserted but not embedded.

## Verifying the setup

After the demo is installed, check that the services are healthy:

```bash
curl http://localhost:4000/health  # oosgql → {"status":"ok"}
curl http://localhost:4100/health  # oosai  → {"status":"ok","embedModel":"..."}
```

Open oos, type _"show me all persons"_ in the chat. The agent should
respond with a board of person records within 2–3 seconds.

## Common issues

**"NATS connection refused"** — `nats-server` is not running or the
URL in settings is wrong. Default is `nats://localhost:4222`.

**"No domains found"** — the internal schema is not installed. Go to
oosd → Demo → Install internal schema.

**Empty boards in oos** — the demo data is not installed, or oosai
could not embed the domains. Check the oosai terminal output for
errors.

**Events not embedded** — oosai was not running when the demo was
installed. Re-run Install demo tables and data with oosai running.

**oosgql schema is stale** — a domain was saved but oosgql did not
rebuild. Check that NATS is running and that oosgql logged
`domain changed — rebuilding schema`.

## Non-default local setup

The defaults assume everything runs on `localhost` with the demo
credentials. A fresh checkout requires no configuration at all in that
case. If your setup differs, copy the `.env.example` files and adjust:

```bash
cp apps/oosgql/.env.example apps/oosgql/.env
cp apps/oosai/.env.example  apps/oosai/.env
```

Then edit the values you need to change. Bun loads `.env` automatically
when you run `bun run dev`. The `.env` files are listed in `.gitignore`
and must never be committed.

Typical reasons to override defaults:

| Situation | Variable to set |
|---|---|
| Postgres on a different host or port | `OOSGQL_PG_URL`, `OOSAI_PG_URL` |
| Postgres with different credentials | same, full URL |
| NATS on a non-standard port | `OOSGQL_NATS_URL`, `OOSAI_NATS_URL` |
| Ollama on a different machine | `OOSAI_EMBED_URL` |
| Commercial embedding API (OpenAI etc.) | `OOSAI_EMBED_URL`, `OOSAI_EMBED_KEY` |
| Running oosgql/oosai on non-default ports | `OOSGQL_PORT`, `OOSAI_PORT` |

> **Note for Kubernetes deployments:** `OOSGQL_HOST` and `OOSAI_HOST`
> must be set to `0.0.0.0` so readiness probes can reach the container.
> The Helm chart and the raw manifests already handle this. Do not set
> these in your local `.env` — `localhost` is correct for local dev.
