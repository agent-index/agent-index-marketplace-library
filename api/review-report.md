---
name: review-report
type: skill
version: 1.2.0
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

Also flag **pointer/ACL drift** (`libptrstale`) as a separate "stale catalog metadata" list: best-effort `aifs_get_permissions` on each in-scope doc's `folder_id` and note any whose real owner/scope no longer matches the pointer — owner changed via a raw `transferOwnership`/`owner_departed`, or `scope` no longer matches the actual grants (`private_shared` but only the owner has access, or `private` but others do). This is the periodic reconciliation sweep that `find-doc` defers to. Read-only — still writes nothing; surface the drift so an admin/owner can heal the pointer (e.g. via `@ai:edit-doc` / `@ai:set-doc-visibility` / `@ai:transfer-doc-ownership`).

Also flag **duplicate and legacy-named pointers** (`libownerxfer`) as a separate "catalog hygiene" list, since only an admin (a Shared-Drive Content Manager) can delete index files:
- **Duplicate pointers:** two or more pointer files sharing the same doc `id` — a legacy artifact of the old `{owner_hash}-{slug}` naming, where an ownership change created a second file the member couldn't trash. List each set, identify which one is ACL-correct (matches `aifs_get_permissions`), and name the stale twin(s) for the admin to delete.
- **Legacy filenames:** pointers still named `{owner_hash}-{slug}.json` rather than the current `{id}-{slug}.json`. These are harmless (located by content) but should be migrated; list them so an admin can rename (create `{id}-{slug}.json` + delete the old) at leisure.

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
