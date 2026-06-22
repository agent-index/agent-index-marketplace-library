---
name: edit-doc
type: task
version: 1.0.0
collection: library
description: Edit an existing library document's body, images, or metadata. Updates the content at its resolved location (public commons write is changelog-attributed; private edits stay in the owner's space), bumps the updated date, optionally refreshes last-reviewed, and keeps the catalog pointer in sync — never writing content-derived metadata for private docs.
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

Edit Document changes an existing doc's body, images, or metadata. It resolves the doc and its scope from the catalog, writes the change at the doc's real location, bumps `updated`, optionally refreshes `last_reviewed`, and keeps the catalog pointer in sync. It does **not** change visibility (use `set-doc-visibility`) or placement (use `move-item`).

A public-doc edit is a commons write and is **changelog-attributed** to the editor. A private-doc edit stays in the owner's space. The strip/enrich rule still holds: editing never writes content-derived metadata (`type`/`tags`/`summary`) into a **private** doc's pointer.

### Inputs
The doc to edit, and the changes (body text, images, title is via rename-item, type, tags, summary for public docs).

---

## Configuration
`library_path`, `library_index_path`, `doc_type_policy`, `doc_types` (from setup responses).

---

## Workflow

### Step 1 — Load configuration, identity, and locate the doc
Read setup responses and local `member-index.json` (`member_hash`, `member_folder_id`). Verify auth. Resolve the named doc via its catalog pointer → `scope` and `location` (commons path for public; `folder_id` in the owner's My Drive for private).

### Step 2 — Authorization
- Public doc → any member may edit (uniform commons write); the edit will be attributed.
- Private doc → only the owner or a granted **collaborator** (writer) may edit. If the caller can't write it: "That doc is private; ask {owner} for write access (`@ai:request-access`)." Halt.

### Step 3 — Apply the change
Read `doc.md`/`meta.json` from the resolved location. Apply the member's edits to `doc.md` and/or `assets/`. For a **public** doc, allow updating `type` (re-validate against `doc-types.json` per policy), `tags`, and `summary` in `meta.json` and the pointer. For a **private** doc, edit body/assets only — do not collect or write `type`/`tags`/`summary` into the pointer.

### Step 4 — Lifecycle and attribution
Set `updated` to now. Ask "Mark this as freshly reviewed?" — if yes, set `last_reviewed = today` and recompute `review_due`. For a public-doc write, append an attribution entry to `meta.json`'s changelog (`{ISO, editor display_name, member_hash, summary-of-change}`); if attribution can't be written, surface it loudly.

### Step 5 — Write content + pointer (same operation)
Write `doc.md`/`meta.json` back at the resolved location, then update the catalog pointer (`updated`, and for public docs the enriched fields). JSON-parse-verify the pointer.

### Step 6 — Confirm
"Updated `{title}`{ — attributed to you in the changelog}. {Marked reviewed; next review due {date}.}"

---

## Directives

- Public edits are uniform-write + changelog-attributed; private edits require owner/collaborator.
- Never write `type`/`tags`/`summary` into a private doc's pointer.
- Title changes go through `rename-item`; visibility through `set-doc-visibility`; placement through `move-item` — edit-doc does none of these.
- Keep content and pointer in sync in one operation; JSON-parse-verify.
- Use `aifs_*` only.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`; preserve edits.
- Private-doc write permission denied for a non-owner/non-collaborator → route to `@ai:request-access`, not member-bootstrap.
- Pointer update fails after content write → surface and retry; doc content is saved but discovery metadata is stale until fixed.
