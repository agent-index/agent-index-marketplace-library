---
name: rename-item
type: task
version: 1.0.0
collection: library
description: Rename a group or a document. Changes the display name/title only — the immutable id is untouched, so cross-doc links and saved references keep resolving. Renaming a still-private doc re-surfaces the reminder that its title is org-public.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/"
writes_to: "/shared/library/, /shared/library-index/"
---

## About This Task

Rename changes the display name of a group or the title of a document. It touches the **name/title only** — the immutable id is untouched, so cross-doc links and saved references keep resolving. Renaming a still-**private** document re-surfaces the reminder that its title is org-public.

### Inputs
The item (group or doc) and the new name/title.

---

## Configuration
`library_path`, `library_index_path` (from setup responses).

---

## Workflow

### Step 1 — Load and resolve
Read setup responses and `member-index.json`; verify auth. Resolve the item: group → `tree.json` node; doc → its pointer (and `scope`).

### Step 2 — Authorization
- Group rename → any member (shared taxonomy).
- Doc rename → public doc: any member; private doc: owner or collaborator only.

### Step 3 — Title-leak reminder (private docs)
If the item is a **private** doc, remind: "Heads up — this doc's title is visible org-wide even though its contents are private. Pick a non-revealing title if it's sensitive." Proceed with the member's chosen title.

### Step 4 — Apply
- **Group:** update `name` in `tree.json`.
- **Doc:** update `title` in the pointer and in the doc's `meta.json`. The slug/`location` and `id` do **not** change (renaming never moves or re-slugs the doc — that keeps links stable).
Write and JSON-parse-verify.

### Step 5 — Confirm
"Renamed to `{new name}`. The id and all existing links are unchanged."

---

## Directives

- Never change ids, slugs, or locations on rename — that's what keeps links durable.
- Always re-surface the title-leak reminder when renaming a private doc.
- Keep `meta.json` and the pointer in sync for doc titles.
- Use `aifs_*` only.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- Private-doc rename by a non-owner/non-collaborator → route to `@ai:request-access`.
- Pointer/`meta.json` divergence after a partial write → surface and reconcile.
