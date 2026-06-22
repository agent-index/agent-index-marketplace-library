---
name: create-group
type: task
version: 1.0.0
collection: library
description: Create a documentation group — a top-level category or a sub-group nested under an existing group, to any depth. Groups form the always-org-public taxonomy; this writes a node to the catalog's tree.json with a new immutable id.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/"
writes_to: "/shared/library-index/"
---

## About This Task

Groups are the library's taxonomy — named categories that can nest to any depth (a top-level group, sub-groups under it, sub-groups under those, and so on with no depth limit). The taxonomy is **always org-public**: every member can see the full tree of group names regardless of who created them. This task adds one group node to the catalog's `tree.json`. It writes no document content — groups are pure structure.

A group gets a new **immutable id** (`grp_…`) at creation; its name and parent can change later (via `rename-item` / `move-item`) without breaking anything that references the id.

### Inputs
The member provides the group name and, for a sub-group, the parent group.

### Outputs
One updated `tree.json` in the catalog with the new group node appended.

---

## Configuration

Read from `collection-setup-responses.md` at runtime:

- **`library_index_path`** — catalog path (default `/shared/library-index/`).

---

## Workflow

### Step 1 — Load configuration and verify access
Read `collection-setup-responses.md` via `aifs_read` for `library_index_path`. Verify auth (`aifs_auth_status`; re-auth on failure; `@ai:member-bootstrap` guidance if unrecoverable).

### Step 2 — Determine placement
Ask whether this is a top-level group or a sub-group.
- Top-level → `parent_id: null`.
- Sub-group → ask which existing group is the parent. Read `tree.json`; resolve the parent by name (disambiguate by showing the path if names collide). If no parent matches, offer to create the parent first or pick from the tree.

### Step 3 — Collect the name
Ask for the group name (e.g. "Server Ops", "Security", "Onboarding FAQs"). Remind the member, if helpful, that organizational domains belong in the tree as groups (not as document *types*).

### Step 4 — Confirm and write
Show the resulting placement ("`Server Ops / Security / {new group}`") and confirm. Then read `tree.json`, append:
```json
{ "id": "grp_{random}", "name": "{name}", "parent_id": "{parent_id_or_null}", "order": {next_order_in_parent} }
```
generating a fresh immutable `grp_` id, and write `tree.json` back via `aifs_write`. JSON-parse-verify the file after writing (no silent corruption).

### Step 5 — Confirm to member
"Created the group `{path}`. You can now file documents under it with '@ai:create-doc' or add sub-groups."

---

## Directives

- Any member may create groups — the taxonomy is a shared org resource.
- Group ids are immutable; never regenerate an id on rename or move.
- Never inline documents into `tree.json`; docs are tracked by their own pointers (which carry `group_id`).
- There is no depth limit — never reject a sub-group for being "too deep."
- Private groups are not supported in v1; if asked, explain that the taxonomy is org-public and suggest private *documents* under a public group.
- Use `aifs_read`/`aifs_write` for all remote operations; JSON-parse-verify `tree.json` after every write.

---

## Error Handling

- Filesystem auth/connectivity failure → halt with `@ai:member-bootstrap` guidance; do not lose the collected name.
- Concurrent-write safety: re-read `tree.json` immediately before appending so a parallel group creation isn't clobbered; if the parse fails, halt rather than overwrite.
- Duplicate name under the same parent → allowed, but warn and confirm (names need not be unique; ids are).
