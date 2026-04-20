---
layout: default
title: Security and roles
nav_order: 8
---

# Security and roles

OOS is built on the premise that access control is not an add-on. This
page walks through how authentication flows through the system, how
authorisation is declared per context, and where each check actually
happens.

## Two layers: who are you, what may you do

**Authentication** happens once, at login. OOS speaks OIDC and uses
PKCE — you are redirected to your identity provider (we develop
against Zitadel), you log in there, and OOS receives an ID token with
your claims. There is no local user database; we are not in the
password business.

**Authorisation** happens on every request. Every context declares
which roles may read it, write it and delete it. Both the client and
the server consult that declaration independently.

## Groups and roles

After the ID token comes back, the client extracts the user's group
memberships from the claims (primary path: the `groups` claim;
fallbacks include Zitadel's project-roles claim and, for the dev
environment, a username mapping).

Groups are mapped to roles in `groups.xml`, stored alongside the
context definitions:

```xml
<oos_groups>
  <group name="oos-admin"   role="admin">
    <include ctx="person.ctx.xml"/>
    <include ctx="note.ctx.xml"/>
  </group>
  <group name="oos-manager" role="manager"> ... </group>
  <group name="oos-user"    role="user">    ... </group>
</oos_groups>
```

One user may be in several groups. The client picks the one with the
highest-priority role (admin > manager > user) and sends that group's
name to oosp as the `X-OOS-Group` header. oosp resolves it back to a
role and uses it to scope every response.

## Context-level permissions

Every context lists which actions each role may perform:

```xml
<permission role="admin"   actions="read,write,delete"/>
<permission role="manager" actions="read,write"/>
<permission role="user"    actions="read"/>
```

Three actions exist:

- **read** — query the context, open the detail screen, see the data.
- **write** — insert or update records.
- **delete** — remove records.

A context with *no* `<permission>` entries is treated as unrestricted.
A context with *any* entry is in deny-by-default mode for every role
not listed.

## Where each check runs

### Client-side: informed UI and informed assistant

The client builds a permission matrix from the AST at session start —
"for your role, here is what you may do in each context". Two things
consume it:

1. The **AI assistant's system prompt** includes the matrix so the
   assistant can give the user early, specific feedback: "your role
   `user` can only read person_list; writing requires `manager` or
   `admin`." This is behaviour, not enforcement.
2. The **UI** uses the same matrix to decide which header buttons to
   render. A user without `delete` permission does not see a delete
   button on the entity screen.

The client's job here is polish. It tells the user what is and is
not possible before they try it. But the client is also untrusted;
nothing above stops a crafted request from reaching the server.

### Server-side: authoritative refusal

Every mutation-class endpoint on oosp resolves the caller's role and
checks it against the context's permission matrix before executing
anything. Two endpoints do this today:

- `POST /save` — the oos_save path. The context name is in the
  request body. The check asks: "does this role have `write` on
  this context?"
- `POST /mutation` — the oos_mutation path. The mutation string
  is parsed to extract the action (insert/update/delete) and the
  context name. `insert` and `update` map to `write`; `delete` maps
  to `delete`. The check asks: "does this role have <action> on
  <context>?"

If the check fails, the request returns HTTP 403 with an `error`
body, and no resolver runs. If the mutation string cannot be parsed,
the request returns 400 — we refuse to execute an unclassifiable
mutation rather than guess.

Read-path permissions (the `/query` endpoint) are not yet gated at
the server level. The client's permission matrix hides the UI
entrances, but a hand-crafted request could still retrieve data.
This is acknowledged; it is on the roadmap.

## Why two layers

Two honest answers:

1. **Usability.** The client knows the user's role immediately and
   can shape the whole experience around it — which buttons to show,
   what the assistant can offer, what to tell the user when they ask
   for something out of reach. If this were server-only, every
   disallowed action would become a round-trip and a red banner.
2. **Trust.** The server cannot believe the client. A modified client
   could claim any role, skip the UI gating, and send any request
   directly. The server doesn't care what the client says it should be
   allowed to do; it re-checks from the authoritative source.

This is a standard pattern, but it's worth stating because it is easy
to pick a side and do only one. Doing only the client means no real
security. Doing only the server means a terrible user experience. Both
together is the point.

## What's next

- [The AI assistant](./the-ai-assistant.html) — how the permission
  matrix feeds into the system prompt.
- [Running OOS](./running-oos.html) — IAM configuration.
