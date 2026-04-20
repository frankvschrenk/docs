---
layout: default
title: Foreword
nav_order: 2
---

# Foreword

We spent a good amount of time in the lab working out where AI genuinely
earns its keep inside a rapid application development (RAD) platform,
and where it does not. The clearest answer we arrived at is that AI
shines at **querying data in natural language**. OOS is built around
that answer: the user asks for what they want, the assistant translates
the request into a grounded query, and the system renders the result.

A second question we wrestled with was how to make a RAD usable inside
real enterprise environments. Most modern cross-platform application
frameworks are, under the hood, browsers — or webviews dressed up as
native apps. Browser-based software is, famously, *insecure by design*:
its threat surface is the open web, not the machine it runs on. Wails,
Tauri and similar webview-based toolkits improve on this but do not
eliminate it.

We decided to bet on a genuinely native path instead. OOS uses **Fyne**,
which in turn renders through **Skia** directly on the GPU. The gain is
not only security — real native windows, multiple windows, predictable
input handling, and none of the JavaScript attack surface — but also
ergonomics: more complex applications become much easier to build when
you can lean on real windowing.

### Security as a first-class property

Not every user should see every record, and not every user should be
allowed to change what they see. OOS is **IAM-first**. Neither the
desktop client (oos) nor the backend (oosp) will do anything until the
user has authenticated.

Authorisation is the second layer. Every domain object — a "context",
in OOS vocabulary — declares which roles may read it, write it and
delete it. This is consulted twice:

- The **client** shows the assistant the current user's role and the
  resolved permission matrix, so the assistant can tell the user
  proactively when an action is out of reach, and so the UI hides
  buttons it should not offer.
- The **backend** enforces the same rules independently. If a client
  bypasses its own gate, oosp still refuses. The client's decisions are
  a kindness; the server's decisions are the truth.

### Describing the world instead of programming it

A RAD platform is valuable in direct proportion to how little code has
to be written. OOS replaces two kinds of code with two kinds of
description:

- **Screens** are described in a compact XML DSL. A form, a table, a
  tab group, a dropdown bound to a meta table — all declared, not
  programmed. A designer can author these directly.
- **Contexts** — the backend-side description of an entity, its
  fields, its relations, its permissions and its filter shape — are
  also XML. From these, OOS builds a GraphQL schema at runtime and
  generates chunks of schema text that the AI assistant reads to ground
  itself in what actually exists.

Both live in the database, not on disk: they can change without a
rebuild, and the GraphQL schema rebuilds itself whenever they do.

### How the AI sees the system

The assistant runs as a ReAct agent with a small set of tools — load
data, preview changes, save, delete, search the schema. Its system
prompt is composed from three sources at session start: who the user
is, what the schema looks like (as embedded chunks or, for larger
deployments, as a vector-searched index powered by a granite embedding
model), and the global prompts authored in `global.conf.xml`.

Crucially, the LLM never sees the data itself — only the schema and
the query results relevant to the user's question, and even those stay
inside the conversation. Sensitive records move between oos and oosp,
not between oosp and any third-party model. This is deliberate, because
the same architecture must work whether the model runs on-premise or
in a hosted service outside the company.

### License

OOS is source-available under a modified **Business Source License
1.1**. You may use it for any purpose inside your company, including
commercial work. You may not resell it or run it as a hosted service
for third parties without a separate agreement. On the Change Date,
the license converts to Apache 2.0. See [license](./license.html) for
the short version and `LICENSE.md` in the repository root for the
binding text.
