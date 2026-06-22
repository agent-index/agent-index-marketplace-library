---
name: archive-doc
type: task
version: 1.0.0
collection: library
description: Soft-retire a document: mark its catalog pointer archived; the content is left intact in the commons (nothing hard-deleted under /shared). A private owner may hard-delete their own doc — it vanishes cleanly if it was never shared, or leaves a revoked record if it ever had a pointer.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
writes_to: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
---

## About This Task

Archive Document soft-retires a doc: it marks the catalog pointer `archived` and leaves the content intact in the commons (nothing is hard-deleted under `/shared`). Archived docs drop out of normal `find-doc`/`browse` results but remain retrievable as archived. A **private** owner may instead hard-delete their own doc — it vanishes cleanly if it was never shared, or leaves a `revoked` record if a pointer ever existed.

### Inputs
The doc to archive (or, for a private owner, to delete).

---

## Configuration
`library_path`, `library_index_path` (from setup responses).

---

## Workflow

### Step 1 — Load, resolve, authorize
Read setup responses and `member-index.json`; verify auth. Resolve the doc via its pointer (`scope`, `location`, `owner`). Authorization: a public doc may be archived by the owner or an admin; a private doc only by its owner.

### Step 2 — Choose the action
- **Archive (default, all docs):** confirm, then set the pointer `status: "archived"` (+ `updated`); leave `doc.md`/`meta.json` in place. Optionally set `meta.json` `status: archived` too.
- **Hard-delete (private docs only, owner choice):** confirm explicitly ("This permanently deletes the content"). Delete the doc folder from the owner's My Drive. If a pointer ever existed (the doc had been shared), overwrite it `scope: "revoked"`, `status: "deleted"` (title residue). If it was never shared (no meaningful pointer history), the pointer may be removed — a clean vanish.

### Step 3 — Write and verify
Write the pointer; JSON-parse-verify. For a hard-delete, confirm the folder is gone.

### Step 4 — Confirm
Archive: "Archived `{title}` — it's out of normal search results but still retrievable. Nothing was deleted." Delete: "Deleted `{title}` from your space." (+ note on any revoked-title residue.)

---

## Directives

- Default is **soft** archive; never hard-delete anything under `/shared`.
- Only a private doc's owner may hard-delete, and only their own doc.
- Ever-shared private docs leave a `revoked` record on delete; never-shared ones may vanish cleanly.
- Keep the pointer authoritative; JSON-parse-verify writes.
- Use `aifs_*` only.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- Unauthorized archive/delete → explain who may do it; make no changes.
- Delete half-fails (folder partly gone) → surface the partial state; reconcile the pointer to reality.
