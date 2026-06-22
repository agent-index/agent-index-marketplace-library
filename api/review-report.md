---
name: review-report
type: skill
version: 1.0.0
collection: library
description: Report on documentation freshness — list documents past their review-due date, optionally scoped to a group subtree or an owner. Covers both public and private docs (review dates are lifecycle metadata, not content). Pairs naturally with a scheduled task.
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

Review Report surfaces documentation staleness — the failure mode that kills wikis. It lists documents whose `review_due` date is in the past, optionally scoped to a group subtree or an owner. Because review dates are **lifecycle** metadata (not content-derived), the report covers both public and private docs. It reads the catalog only and writes nothing. It pairs naturally with a scheduled task ("every Monday, list overdue docs").

---

## Configuration
`library_index_path`, `library_default_review_cadence_days` (from setup responses).

---

## Workflow

### Step 1 — Load
Read setup responses; verify auth. Read `tree.json` and the per-doc pointers.

### Step 2 — Scope (optional)
If the member named a group ("review report for Server Ops") → restrict to that subtree. If they named an owner ("what of mine is stale") → restrict to that owner. Otherwise → whole library.

### Step 3 — Compute overdue
For each pointer in scope, compare `review_due` to today. Collect those past due (and optionally those due within N days as "due soon"). Skip `archived`/`revoked` docs unless asked.

### Step 4 — Report
Present overdue docs grouped by group path (or by owner), each with: title, owner, `last_reviewed`, and how overdue. Sort most-overdue first. Note the count. Offer next steps: open `@ai:edit-doc` to refresh a doc's review date, or reassign an owner.

### Step 5 — Offer to schedule
If not already scheduled, offer: "Want me to run this automatically — say, every Monday morning — and flag what's overdue?"

---

## Directives

- Covers public AND private docs (staleness is not content; only lifecycle dates are read).
- Read-only; this skill writes nothing.
- Respect scope filters (group subtree / owner) when given.
- Use `aifs_*` reads only.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- A pointer missing `review_due`/`last_reviewed` → treat as "review date unknown," list it separately rather than crashing.
