---
name: manage-doc-types
type: task
version: 1.0.0
collection: library
description: Admin capability to govern the document-type vocabulary: flip the policy between locked (admin-approved types only) and open (members may add types at creation), and add or remove types. Removing a type that is in use warns and lists affected docs rather than rewriting them.
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

Manage Doc Types governs the document-type vocabulary in `doc-types.json`. It is **admin-only**. An admin can flip the **policy** between `locked` (only admin-approved types may be used) and `open` (any member may introduce a new type at doc creation, which is then registered org-wide), and can **add** or **remove** types. Removing a type that is in use warns and lists affected docs — it does not rewrite them.

### Inputs
The action: set policy (`locked`/`open`), add type(s), remove type(s), or list.

---

## Configuration
`library_index_path`, `doc_type_policy`, `doc_types` (from setup responses; `doc-types.json` is authoritative at runtime).

---

## Workflow

### Step 1 — Load and authorize
Read setup responses and `org-config.json`; resolve the caller's org role from `member-index.json` / registry. **Admin-only** — if the caller isn't an admin: "Managing doc types is an admin action. Ask an admin, or (if your org allows it) ask an admin to switch the doc-type policy to ‘open’ so members can add types at creation." Halt. Verify auth. Read `{library_index_path}/doc-types.json`.

### Step 2 — Choose the action
List / set-policy / add-type / remove-type.

### Step 3 — Apply
- **List:** show `policy` and `types`.
- **Set policy:** set `policy` to `locked` or `open`. Explain the effect ("open lets any member add a new type when creating a doc").
- **Add type(s):** append to `types` (dedupe; preserve case as entered).
- **Remove type(s):** before removing, scan the per-doc pointers for docs using that `type`; if any, **warn** and list them ("3 docs use ‘Postmortem’; they'll keep that label until re-typed"). Confirm, then remove from `types`. Do **not** rewrite the affected docs.

### Step 4 — Write and verify
Write `doc-types.json`; JSON-parse-verify.

### Step 5 — Confirm
"Doc-type policy is now `{policy}`. Types: {list}." (+ any in-use warnings for removals.)

---

## Directives

- Admin-only for all mutations. (Members add types only indirectly, at doc creation, and only when `policy: "open"`.)
- Removing an in-use type warns + lists affected docs; never silently rewrites doc metadata.
- `type` describes document FORMAT; if an admin tries to add an organizational domain (e.g. "IT"), suggest a group instead.
- Use `aifs_*` only; JSON-parse-verify `doc-types.json` after writing.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- Non-admin caller → halt with the explanation above; make no changes.
- Concurrent edit: re-read `doc-types.json` immediately before writing to avoid clobbering a parallel change.
