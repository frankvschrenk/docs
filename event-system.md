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
`public.event_type_grammar` and are edited in oosd's **Event Types**
panel with Monaco and the `onisin-event-schema` language.

Example grammar entry names: `Einbruch`, `Geiselnahme`, `SupportTicket`.

**Event mapping** connects a source table in the application database
to the event system. A mapping says: _"the `police_incidents` table is
an event source"_. Mappings live in `public.event_mappings`. Each mapping
has a `tag` that provides context for the streams belonging to it.

**Event stream** is a named context within a mapping — typically a case
file, a session, a job or a thread. Streams live in `public.event_streams`.
Example stream IDs: `fall-2024-0042`, `customer-12345`.

**Event** is one row in the source table. When an event is inserted
(via `oos.cmd.event.insert`), oosai embeds the event text and stores
the vector in the mapping's embeddings table. This makes all events
semantically searchable.

## Data model

The registry tables live in the `public` schema so they sit next to
the user-owned source tables they point at:

```
public.event_mappings
  id            — mapping identifier (e.g. "police", "support")
  event_types   — jsonb array of accepted grammar names
  tag           — context label for streams (e.g. "fall", "ticket")
  close_policy  — default close behaviour: 'manual' | 'auto_on_insert'

public.event_streams
  stream        — stream identifier (PRIMARY KEY)
  mapping_id    — FK to event_mappings
  tag           — per-stream context

public.event_type_grammar
  name          — grammar name (UNIQUE, e.g. "Einbruch")
  source        — DSL source text
  tags          — jsonb array of context tags
  close_policy  — override for this type: 'manual' | 'auto_on_insert' | NULL (inherit)
```

Each source table (e.g. `public.police_incidents`,
`public.support_tickets`) carries two lifecycle columns:

```
closed        boolean     NOT NULL DEFAULT false
closed_at     timestamptz NULL
```

A `BEFORE UPDATE OR DELETE` trigger on every source table enforces
the immutability rule described in [Event lifecycle](#event-lifecycle).

## Authoring event types (oosd)

The **Event Types** panel in oosd shows all grammars in a list. Select
one to open its source in Monaco. Save with `Cmd+S`.

Grammars are a grammar-library independent of any specific mapping —
one grammar can be accepted by multiple mappings.

## Assigning event types to mappings (oosd)

The **Mapping Types** panel shows a checklist of all grammars for each
mapping. Check or uncheck to add or remove a grammar from a mapping's
`event_types` array. This writes to `public.event_mappings.event_types`
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
  payload:    object   // arbitrary structured data
}
reply:   { ok: true, id: number, closed: boolean }
```

The handler resolves the effective close policy (see
[Close-policy resolution](#close-policy-resolution)) before writing
the row. If the policy is `auto_on_insert`, the row is written with
`closed = true` and `closed_at = now()` in the same INSERT, so the
immutability trigger applies from the first moment. The reply tells
the caller whether the new row is already closed.

After the INSERT, the handler publishes the same payload (plus an
`op: "insert"` field) on `oos.events.<mapping>`. oosai's embedding
listener picks it up, embeds the `text` field, and upserts the vector
together with the metadata into the mapping's vector table.

## Event lifecycle

Events have two states: **open** and **closed**. New events are open
by default; once an event is closed it becomes immutable.

### Why two states

Many event sources are append-only in spirit but not yet final in
fact. A police incident report needs corrections while the officer
is still on scene. A support ticket gets edits while the agent is
still on the phone. Once the incident is filed or the ticket is
resolved, the record should never change again — that's the
auditability promise downstream consumers (and the LLM RAG
lookup) rely on.

The `closed` flag captures that distinction. Open events can be
edited, re-embedded, or removed entirely. Closed events are frozen.

### Immutability trigger

Every source table installs a `BEFORE UPDATE OR DELETE` trigger
that raises `check_violation` (SQLSTATE `23514`) when `OLD.closed`
is `true`:

```sql
CREATE OR REPLACE FUNCTION public.prevent_modify_closed_event()
RETURNS TRIGGER LANGUAGE plpgsql AS $func$
BEGIN
    IF OLD.closed = true THEN
        RAISE EXCEPTION 'event % is closed and cannot be %', OLD.id, lower(TG_OP)
            USING ERRCODE = '23514';
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$func$;
```

The trigger fires on `OLD.closed`, so the transition
`open → closed` itself is allowed (because the row was open at the
start of the statement). Every later UPDATE or DELETE on the same
row rolls back.

The rule is one-way and absolute: there is no "reopen" operation.
If a closed event is wrong, the correction is a new event that
references the original.

### Close-policy resolution

`close_policy` controls whether new events start open or are closed
on insert. It is resolved at insert time with this precedence:

| Layer                  | Column           | Values                                   |
|------------------------|------------------|------------------------------------------|
| Event type grammar     | `close_policy`   | `'manual'` &#124; `'auto_on_insert'` &#124; `NULL` (inherit) |
| Event mapping (fallback) | `close_policy` | `'manual'` &#124; `'auto_on_insert'`     |
| Hard default           | —                | `'manual'`                               |

Resolution order:

1. Look up `event_type_grammar.close_policy` for the inserted event
   type. If non-NULL, that value wins.
2. Otherwise fall through to the mapping's `close_policy`.
3. If the mapping has none either (shouldn't happen — the column
   is `NOT NULL DEFAULT 'manual'`), `'manual'` is the hard default.

`'manual'` means: insert with `closed = false`. The caller closes
the event explicitly later via `oos.cmd.event.close`.

`'auto_on_insert'` means: insert with `closed = true` and
`closed_at = now()`. The event is frozen the moment it is created.
Useful for sources that publish already-final records (e.g. log
entries, audit trails) where no editing window is needed.

## Modifying and closing events

Three additional subjects cover the lifecycle operations beyond
insert. All of them resolve the mapping by name, then operate on
the per-mapping source table.

### Update

```
subject: oos.cmd.event.update
payload: { mapping: string, id: number, text: string, payload: object }
reply:   { ok: true }
```

Updates `text` and `payload` on an open event. `event_type` and
`stream` are intentionally immutable — changing them would
invalidate downstream tags and cross-references.

The trigger blocks updates on closed rows at the DB level; the
`23514` error propagates to the caller. After a successful UPDATE
the handler republishes the new payload on `oos.events.<mapping>`
with `op: "update"`, so the embedding listener re-embeds the row
(same `source_id`, overwrites the vector slot).

### Delete

```
subject: oos.cmd.event.delete
payload: { mapping: string, id: number }
reply:   { ok: true }
```

Deletes an open event row. The trigger blocks deletes on closed
rows. After a successful DELETE the handler publishes
`{ op: "delete", id }` on `oos.events.<mapping>`, and the embedding
listener removes the corresponding vector.

### Close

```
subject: oos.cmd.event.close
payload: { mapping: string, id: number }
reply:   { ok: true, closed_at: string }
```

Flips an open event to closed. This is the one transition the
trigger allows (because `OLD.closed` is still `false` when the
statement begins). After this returns, the row is frozen.

Close is a metadata-only flip; no embedding roundtrip is needed.
The vector already in the embeddings table stays valid.

### Op dispatch on the embedding bus

The NATS subject `oos.events.<mapping>` carries every lifecycle
message to the embedding listener. The `op` field selects the
branch:

| `op`       | Listener action                                      |
|------------|------------------------------------------------------|
| `insert`   | Embed `text`, upsert vector keyed by `source_id`     |
| `update`   | Re-embed `text`, overwrite the same `source_id` slot |
| `delete`   | Remove the embedding row by `source_id`              |

Missing `op` defaults to `insert` so legacy publishers keep working.

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

Closed events render with a lock icon. Right-clicking a closed event
in the history list disables the **Edit** and **Delete** menu items —
the trigger would reject those operations at the DB level anyway, and
the UI surfaces the rule before the round-trip.

Tag-filtered event type grammars for the stream are loaded via
`oos.cmd.event.schemas`.

## Stream deletion

Deleting a stream cascades through its source rows and their
embeddings before the `event_streams` row itself is removed:

```
subject: oos.cmd.event_streams.delete
payload: { stream: string }
reply:   { ok: true }
```

The handler:

1. Resolves the stream's mapping (so it knows which source table
   and which embedding table the events live in).
2. **Probes for closed events** — if any source row for this stream
   has `closed = true`, the whole operation fails with a clear
   error. Closed events block the cascade because the trigger
   would roll back any DELETE attempt anyway, and probing first
   gives a faster, friendlier error than letting the trigger fire.
3. Removes the per-row embeddings via `deleteEventVector`.
4. Deletes the source rows.
5. Deletes the `event_streams` row.

Force-delete is not exposed. Closed means closed — if a stream
contains closed history that must go, the answer is to archive the
stream out of band, not to break the immutability guarantee.

## Boot and backfill

When oosai starts, it:

1. Loads all enabled event mappings from `public.event_mappings`.
2. Subscribes to one NATS subject (`oos.events.<mapping>`) per
   mapping for incoming events.
3. Runs a backfill — embeds any rows in source tables that have
   not yet been processed.

If `public.event_mappings` does not yet exist (fresh DB before the
first seed), the listener treats the missing table as an empty
mapping list and starts idle. A later `oos.cmd.event.refresh` —
typically sent by oosd after installing the internal schema and
seeding a demo — re-reads the table and subscribes to the newly
appeared mappings without an oosai restart. The same refresh path
handles mappings added or removed at runtime.

The recommended startup order for a fresh database is still:

1. `nats-server`
2. `oosgql`
3. `oosai`
4. oosd → **Settings → Install schema** (creates `oos.*` and
   `public.event_mappings`)
5. oosd → **Demo** → Install demo tables and data, then install
   the police or support event source

After step 5, oosd sends `oos.cmd.event.refresh` and oosai picks up
the newly installed mappings.
