---
name: move-item
type: task
version: 1.0.0
collection: library
description: Move a document to a different group, or reparent a group within the taxonomy, to any depth. Reparenting is catalog-logical (the immutable id never changes, so links survive); for public docs the commons folder is relocated to keep the projection honest. Runs a cycle check so a group can't become its own descendant.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library/, /shared/library-index/"
writes_to: "/shared/library/, /shared/library-index/"
---

## About This Task

Move / Reparent changes where something sits in the taxonomy — move a **document** to a different group, or move a **group** (and everything under it) to a different parent, to any depth. Placement is **catalog-logical**: a doc's pointer carries its `group_id`; a group's node carries its `parent_id`. The immutable id never changes, so links and saved references survive a move. For a **public** document, the commons folder path mirrors the tree, so the move also relocates the commons folder to keep that projection honest. A **private** document only updates its `group_id` (its content stays in the owner's My Drive).

### Inputs
The item to move (doc or group) and the new parent group.

---

## Configuration
`library_path`, `library_index_path` (from setup responses).

---

## Workflow

### Step 1 — Load and resolve
Read setup responses and `member-index.json`. Verify auth. Identify whether the member is moving a doc or a group, and resolve it (doc → its pointer; group → its `tree.json` node). Resolve the destination parent group from `tree.json`.

### Step 2 — Validate the move
- **Group move:** run the **cycle check** — the destination parent must not be the group itself or any of its descendants. Reject with a clear message if it would create a cycle.
- **Doc move:** destination group must exist.

### Step 3 — Apply
- **Doc, private:** set the pointer's `group_id` to the destination; no content relocation.
- **Doc, public:** set the pointer's `group_id`, then relocate the commons folder from `{library_path}/{old-path}/{slug}/` to `{library_path}/{new-path}/{slug}/` via `aifs_copy` + delete-source (or move), and update the pointer's `location` path. Verify the new location before removing the old.
- **Group:** set the node's `parent_id` (and `order`) in `tree.json`. Child docs reference the group by `group_id`, so they move logically for free; for public docs whose commons path includes the group name, relocate their folders to match (or, simpler and id-stable: keep folders flat-by-slug under the commons and treat the tree purely logically — follow whichever the implementation chose; the pointer `location` is authoritative either way).

### Step 4 — Write and verify
Write `tree.json` and/or the affected pointer(s); JSON-parse-verify. For relocations, confirm the destination exists and the source is gone.

### Step 5 — Confirm
"Moved `{name}` to `{new path}`. Ids and links are unchanged."

---

## Directives

- Ids are immutable across moves; never regenerate them.
- Always run the cycle check on group moves.
- The pointer `location` (and `tree.json`) is the source of truth; for public docs keep the commons folder consistent with it.
- A private doc's content never moves on a reparent — only its `group_id`.
- Use `aifs_*` only; verify destination before deleting source on any relocation.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- Cycle detected → reject and explain; make no changes.
- Relocation half-fails (copied but source not removed, or vice-versa) → surface the partial state honestly; do not update the pointer to a location that doesn't exist.
