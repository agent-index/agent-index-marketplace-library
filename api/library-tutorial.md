---
name: library-tutorial
type: skill
version: 1.0.0
collection: library
description: Explain how the library works: the always-org-public taxonomy, public-by-default documents, how to go private and share, the rule that private-doc TITLES stay org-public, the governed doc-type vocabulary, staleness/review tracking, and how to find and request access to docs.
stateful: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/"
writes_to: null
---

## About This Skill

Library Tutorial explains how the documentation library works. It's the onboarding capability — run it when a member asks "how does the library work" or is new to the collection. It must teach the privacy model clearly, especially the rule that **private-document titles stay org-public**.

---

## Configuration
None required (optionally read setup responses to show the org's actual defaults).

---

## Workflow

Walk the member through, conversationally:

### 1 — The shape: groups and docs
The library is a tree of **groups** (named categories) that nest to any depth, with **documents** filed under them. Create structure with `@ai:create-group`, file docs with `@ai:create-doc`. Organizational domains (IT, Server Ops, Customer Support) are **groups**; document **type** (HowTo, FAQ, Tutorial, Runbook, …) is a separate label.

### 2 — Everyone sees the map, not always the contents
The **taxonomy and every document title are org-public** — anyone can see what groups and docs exist. Whether you can open a doc's **contents** depends on that doc's visibility.

### 3 — Public by default; private when you choose
Docs are **org-public by default** (everyone can read, and collaboratively edit, in the commons). You can make a doc **private** at creation or any time (`@ai:set-doc-visibility`) — then only you and people you explicitly grant can read it.

### 4 — The important catch: private titles are still public
**A private doc's title stays visible to the whole org** — only its contents are hidden. So if a title itself is sensitive ("Acme Layoff Plan"), **give it a codename**. The system warns you about this every time you go private.

### 5 — Finding things
Ask in plain language — `@ai:find-doc` ("how do I rotate API keys?") searches the catalog and summarizes the right docs; `@ai:browse-library` shows the tree. If you can see a private title but not its contents, `@ai:request-access` pings the owner via your org's comms channel.

### 6 — Keeping docs fresh
Every doc has an owner and a review date (default {cadence} days). `@ai:review-report` lists what's overdue — and can run on a schedule.

### 7 — Doc types
Types describe format and come from a governed list. Depending on org policy, types are either admin-managed (`locked`) or member-extensible (`open`) — admins manage this with `@ai:manage-doc-types`.

End by asking what they'd like to do first (create a group, add a doc, or find something).

---

## Directives

- Always teach the private-title rule (point 4) — it's the easiest mistake to make.
- Keep it conversational and short; offer to kick off a concrete first action.
- If setup responses are readable, show the org's real defaults (visibility, cadence, type policy).
- Read-only; this skill writes nothing.

---

## Error Handling

- If the catalog/setup can't be read, still explain the model from this tutorial; note that live defaults couldn't be loaded.
