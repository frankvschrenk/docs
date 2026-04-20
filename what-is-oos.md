---
layout: default
title: What is onisin OS
nav_order: 3
---

# What is OOS

**onisin OS** is a platform for building data-driven enterprise
applications without writing application code. You describe your data,
you describe how it should look on screen, and OOS does the rest — it
builds the GraphQL schema, renders the UI, and hands an AI assistant
the context to help your users explore and edit that data in natural
language.

## In a single sentence

OOS turns a handful of XML definitions into a running, native,
AI-assisted business application.

## What you get out of the box

- A **native desktop client** (oos) built on Fyne. Not a webview. Real
  windows, real menus, real keyboard handling.
- A **backend server** (oosp) that exposes your data as GraphQL,
  handles permissions, serves the AI assistant's tools, and keeps the
  schema up to date as your context files evolve.
- An **AI assistant** that can read the schema, run queries and
  mutations, and stays grounded — it never invents a field name or a
  query shape.
- A **design tool** (ooso) for authoring context and screen
  definitions, with a live preview.
- A **role-based permission system** wired through both the client's
  UI and the server's handlers.

## What you describe, what OOS generates

You write:

- **Context files** — one per business entity. "Here is what a Person
  is, which fields it has, which roles may read, write and delete it."
- **Screen files** — one per view. "Here is how a Person detail page
  looks: a header, three tabs, this form layout, that dropdown bound
  to those options."
- **Global prompts** — a small XML file that tells the AI assistant
  how to behave: use these tools, don't invent field names, tell the
  user when a role is not allowed to do something.

OOS generates:

- A **GraphQL schema** with queries, filters, inserts, updates and
  deletes for every entity.
- A **retrievable schema index** — text chunks per context, embedded
  into a vector store — so the assistant can find the right context
  for any natural-language request.
- A **Fyne UI** rendered live from your screen definitions, complete
  with bindings, dropdowns that load their options from your
  database, and navigation between views.
- **Permission enforcement** on both sides of the wire.

## Who is it for

OOS is aimed at teams who build internal and line-of-business
applications — the kind of systems where the domain is genuinely
complex but the UI patterns are fairly standard: lists, detail forms,
relations, dropdowns, write-with-confirmation. If you are building
Gmail, this is not the tool. If you are building the software that
keeps a company's operations running, it is.

## What's next

- [How it works](./how-it-works.html) — the architecture and where each
  piece sits.
- [Writing contexts](./writing-contexts.html) — your first context
  definition.
- [Running OOS](./running-oos.html) — installation and seed.
