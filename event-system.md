---
layout: default
title: The event system
nav_order: 6
---

# The event system

The event system lets OOS ingest, classify and semantically search
real-world event records — incident reports, support tickets, log
entries, or any other append-only stream of facts.

## Core concepts

**Event type grammar** is a named DSL source that describes one class
of event: its required and optional fields, its context tags, and
examples of what the event looks like. Grammars live in
`oos.event_type_grammar` and are edited in oosd's **Event Types**
panel with Monaco and the `onisin-event-schema` language.

Example grammar entry names: `Einbruch`, `Geiselnahme`, `SupportTicket`.

**Event mapping** connects a source table in the application database
to the event system. A mapping says: _"the `police_incidents` table is
an event source"_. Mappings live in `oos.event_mappings`. Each mapping
has a `tag` that provides context for the streams belonging to it.

**Event stream** is a named context within a mapping — typically a case
file, a session, a job or a thread. Streams live in `oos.event_streams`.
Example stream IDs: `fall-2024-0042`, `customer-12345`.

**Event** is one row in the source table. When an event is inserted
(via `oos.cmd.event.insert`), oosai embeds the event text and stores
the vector in the mapping's embeddings table. This makes all events
semantically searchable.

## Data model

```
oos.event_mappings
  id         — mapping identifier (e.g. "police", "support")
  event_types — jsonb array of accepted grammar names
  tag        — context label for streams (e.g. "fall", "ticket")

oos.event_streams
  stream     — stream identifier (PRIMARY KEY)
  mapping_id — FK to event_mappings
  tag        — per-stream context

oos.event_type_grammar
  name       — grammar name (UNIQUE, e.g. "Einbruch")
  source     — DSL source text
  tags       — jsonb array of context tags
```

## Authoring event types (oosd)

The **Event Types** panel in oosd shows all grammars in a list. Select
one to open its source in Monaco. Save with `Cmd+S`.

Grammars are a grammar-library independent of any specific mapping —
one grammar can be accepted by multiple mappings.

## Assigning event types to mappings (oosd)

The **Mapping Types** panel shows a checklist of all grammars for each
mapping. Check or uncheck to add or remove a grammar from a mapping's
`event_types` array. This writes to `oos.event_mappings.event_types`
via `oos.cmd.event_mappings.set_types`.

## Inserting events

Events are inserted via NATS:

```
subject: oos.cmd.event.insert
payload: {
  mapping:    string   // mapping id
  stream:     string   // stream id
  event_type: string   // grammar name
  text:       string   // human-readable event text to embed
  metadata:   object   // arbitrary structured data
}
```

oosai embeds the `text` field and stores the vector together with the
metadata in the mapping's vector table. The event is then searchable
by semantic similarity.

## Searching events (oos)

In oos, switch to **Events** mode. Select a stream from the picker and
type a question. The agent calls `oos.cmd.event.search` with the query
embedded as a vector. oosai returns the most similar events. The agent
builds a grounded answer from those results without using GraphQL.

The stream picker is required. Without a stream ID, the context would
be too broad for reliable answers.

## Stream detail (oosd)

Right-click a stream in oosd → **Editieren** opens a `stream_detail`
tab. The left side shows the event history for that stream. The right
side has an event type autocomplete and a Monaco editor for composing
new events.

Tag-filtered event type grammars for the stream are loaded via
`oos.cmd.event.schemas`.

## Boot and backfill

When oosai starts, it:

1. Loads all active event mappings from `oos.event_mappings`.
2. Subscribes to one NATS subject per mapping for incoming events.
3. Runs a backfill — embeds any rows in source tables that have not
   yet been processed (`processed = false`).

If oosai starts before any mappings exist, the listeners go idle.
They do not auto-recover when mappings are added later. The safe
startup order is:

1. `nats-server`
2. `oosgql`
3. `oosai`
4. oosd → Demo → Install internal schema
5. oosd → Demo → Install demo tables and data

After step 5, all mappings are present before oosai's command handler
becomes active.
