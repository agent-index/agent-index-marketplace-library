---
name: browse-library
type: skill
version: 1.0.0
collection: library
description: Render the documentation taxonomy — groups, unlimited-depth sub-groups, and document titles — joined from the always-public catalog. Titles only, zero content reads. Shows a scope badge per doc and offers request-access for private docs the member can't read.
stateful: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/, /shared/library/"
writes_to: null
---

## About This Skill

Browse Library renders the documentation taxonomy — groups, unlimited-depth sub-groups, and the **titles** of the documents in them — from the always-public catalog. It reads **titles only** and never opens document content. Every member sees the full tree (the taxonomy is org-public); each doc shows a scope badge, and private docs the member can't read still appear by title with a request-access affordance.

---

## Configuration
`library_index_path` (from setup responses).

---

## Workflow

### Step 1 — Load
Read setup responses for `library_index_path`. Verify auth. Read `tree.json` and the per-doc pointers from the catalog.

### Step 2 — Resolve the view
- No scope given → render from the top-level groups down.
- "What's under {group}" → resolve that group and render its subtree.

### Step 3 — Render (titles only)
For each group, list sub-groups (recursively, no depth limit) and the docs whose `group_id` matches, showing for each doc: **title**, doc **type** (public docs only), and a **scope badge**:
- `org-public` — readable by all.
- `shared with you` — private but you're granted.
- `private` — not readable by you; show title + owner + "ask for access (`@ai:request-access`)".
Do **not** open any `doc.md`. Do not infer private content from titles.

### Step 4 — Offer next steps
Suggest `@ai:find-doc` for topic search, or opening a specific readable doc.

---

## Directives

- Titles only — never read document bodies in this skill.
- Show the whole taxonomy regardless of content permissions (it's org-public).
- For private docs the member can't read, surface title + owner + request-access; never the body.
- Render arbitrary depth; never truncate the tree for being "too deep" (offer to focus on a subtree if large).
- Use `aifs_*` reads only; this skill writes nothing.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- A malformed pointer → skip it in the render and note "1 catalog entry couldn't be read" rather than failing the whole view.
