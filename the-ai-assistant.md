---
layout: default
title: The AI assistant
nav_order: 5
---

# The AI assistant

OOS ships two AI agents: one in **oos** for end users querying data,
and one in **oosd** for developers authoring DSL files. Both run the
same ReAct loop pattern but with different tool sets and system prompts.

## The ReAct loop

The loop is intentionally small — roughly 80 lines, no framework. The
shape is the textbook one:

```
loop (max 8 steps):
  ask LLM
    → text response  → done, return to user
    → tool call(s)   → run tools, append results, loop again
```

`MAX_STEPS = 8` keeps a confused model from spinning. A typical query
takes 2 steps (schema search + data query). A join takes 3 or 4.
Beyond 8 steps something is wrong with the prompt, not the loop.

Tool errors are returned to the LLM as JSON result payloads. The
model is expected to recover — this is how it corrects hallucinated
field names.

## The system prompt (oos)

The prompt is built fresh on every turn from three layers:

**1. Global standing instructions**
Loaded from oosai (`oos.cmd.global`). Every row in `oos.global_prompt`
is a named instruction block — deletion rules, filter conventions,
response style, output formatting. Change a row in the database and the
LLM's behaviour changes on the next message, without restarting anything.

**2. Domain index**
One line per known domain: name and scope sentence. This gives the LLM
an overview of all available entities before it starts searching.

**3. Worked examples**
A small set of concrete query-and-answer examples that teach the LLM
the expected tool call pattern. Injected once per turn.

This design is called the **knowledge sandwich**: always-present global
instructions and domain index sandwich the user message, with on-demand
schema chunks arriving in the middle as tool results.

### With a view hint

When oosd's local MiniLM resolver has already identified the target
domain and view, oos can skip the `oos_schema_search` step. The system
prompt receives the view's field list directly and instructs the model
to use that domain. The worked examples are omitted because the
discovery work is already done.

## Tools (oos)

| Tool | NATS subject | Description |
|---|---|---|
| `oos_schema_search` | `oos.cmd.search` | Cosine search over embedded domain chunks. Returns the best matching context with all field and filter information. |
| `oos_query` | `oos.cmd.gql.query` | Execute a GraphQL query. The LLM builds the query string from the schema chunk returned by `oos_schema_search`. |

The LLM never constructs mutations directly. Mutations go through a
separate confirmation flow in the UI.

## Tools (oosd agent)

The oosd agent has a different tool set focused on DSL authoring:

| Tool | Description |
|---|---|
| `domain_list` | List all domains by name |
| `domain_load` | Load the DSL source of one domain |
| `view_save` | Save a view DSL source to the database |
| `domain_save` | Save a domain DSL source to the database |

When the agent produces a DSL code block in its response, oosd
extracts it and opens it directly in the Monaco editor.

## OpenAI-compatible API

Both agents use the OpenAI chat completions API shape. Any
OpenAI-compatible provider works: Ollama (default for local
development), OpenAI, Anthropic via proxy, or any other compatible
endpoint. Configure the base URL, API key and model name in the
respective settings panel.

## Embedding model

Domain and event embeddings use `granite-embedding:latest` via Ollama
by default. The oosd local resolver uses
`paraphrase-multilingual-MiniLM-L12-v2` running in-process in the
renderer (no network call, works offline).

## Behaviour through data, not code

The most important design principle: **LLM behaviour is controlled by
data in the database, not by code changes**.

- Add a row to `oos.global_prompt` → the LLM sees a new instruction on
  the next message.
- Edit a domain's `ai` hints → the LLM gets updated context on the next
  schema search.
- Change the model name in settings → the next turn uses the new model.

No redeployment, no restart.
