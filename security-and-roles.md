---
layout: default
title: Security and roles
nav_order: 8
---

# Security and roles

## Permission declarations

Every domain declares which roles may perform which actions:

```
domain person from person@demo {
  permission admin   read, write, delete
  permission manager read, write
  permission user    read
  ...
}
```

Supported actions: `read`, `write`, `delete`.

## Where permissions are enforced

Permissions are enforced at two levels:

**System prompt** — the AI agent's system prompt includes the
permission matrix for the current user's role. The LLM knows ahead
of time what the user may do and will decline to propose mutations
that exceed the user's rights. This is a convenience layer — it
prevents the assistant from proposing operations that would fail.

**GraphQL layer (oosgql)** — every query and mutation is checked
against the permission matrix before execution. The check is
performed server-side by `packages/oos-gql-ts`. If the user's role
does not have the required action on the requested domain, the
request is rejected with an error — even if the client bypassed the
LLM layer entirely.

The client-side and LLM-side permission awareness are therefore
defence-in-depth. Compromising the client does not grant elevated
database access.

## Role resolution

The user's role is resolved from the `X-OOS-Group` header on every
request. In development, the demo falls back to a username mapping:
`admin` → `oos-admin`, `user` → `oos-user`. For production, set the
groups claim from your identity provider.

## The permission matrix in the LLM prompt

The `oos.global_prompt` table carries standing instructions for the
LLM, including role-specific behaviour rules. For example, a prompt
named `deletion` might instruct the LLM to always ask for confirmation
before proposing a delete, and to only offer deletion to users whose
role includes the `delete` action.

These instructions are data — edit a row in `oos.global_prompt` and
the LLM's behaviour changes on the next message without restarting
anything.

## What OOS does not (yet) provide

- OIDC / SSO integration — there is no built-in identity provider.
  Authentication is expected to be handled by a reverse proxy or
  gateway that sets the `X-OOS-Group` header.
- Row-level security — permissions are per-domain, not per-row.
  If a role has `read` on `person`, it can read all persons.
- Vault / mTLS per-session credentials — planned for a future release.
