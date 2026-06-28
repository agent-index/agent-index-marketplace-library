---
name: create-doc
type: task
version: 1.1.0
collection: library
description: Create a document in the library — org-public by default (the /shared/library/ commons) or private (the owner's My Drive, content gated to owner + grants). Files the doc under a chosen group, validates its type against the governed vocabulary, and writes the always-public catalog pointer (enriched metadata for public docs only). Going private surfaces the title-leak warning.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/"
writes_to: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
---

## About This Task

Create Document files a new document in the library. A document is a **folder** (`doc.md` + `meta.json` + an `assets/` subfolder for images/attachments), so docs can carry images. The doc is **org-public by default** — living in the `/shared/library/` commons where every member can read and collaboratively edit it — or **private**, living in the owner's My Drive and readable only by the owner and members they later grant.

Two things this task must get right, every time:

1. **The catalog pointer is written in the same operation as the content.** The always-public catalog (`/shared/library-index/`) is the source of truth for what exists; a content write whose pointer write fails must be surfaced and retried, never left silently un-indexed.
2. **Enriched metadata (type, tags, summary) goes in the pointer only for PUBLIC docs.** A private doc's pointer carries title + placement + owner + lifecycle dates **only** — never a summary, tags, or anything content-derived (decision 6). And a private doc's **title is still org-public** — so creating private triggers the title-leak warning (decision 5).

### Inputs
Group placement, title, body (and any images), doc type, and the visibility choice.

### Outputs
- Public: `{library_path}/{group-path}/{doc-slug}/` (`doc.md`, `meta.json`, `assets/`) + a catalog pointer with enriched fields.
- Private: `id:{member_folder_id}/library/{doc-slug}/` (same layout) + a catalog pointer **without** enriched fields.

---

## Configuration

Read from `collection-setup-responses.md`:

- **`library_path`** (default `/shared/library/`), **`library_index_path`** (default `/shared/library-index/`)
- **`library_default_visibility`** — `org_public_first` | `ask` | `private_first`
- **`library_default_review_cadence_days`** (default `180`)
- **`doc_type_policy`** (`locked` | `open`) and **`doc_types`** (the vocabulary)

---

## Workflow

### Step 1 — Load configuration and identity
Read setup responses for the parameters above. Read local `member-index.json` for `member_hash` and **`member_folder_id`** (missing → "Your private member space isn't set up — run `@ai:update` and retry." Halt). Verify auth.

### Step 2 — Placement
Ask which group the doc belongs under (read `tree.json`; resolve by name/path; offer `create-group` if the target doesn't exist yet). Record the `group_id`.

### Step 3 — Title and body
Collect the title and the document body (markdown). If the member has images to include, collect them for the `assets/` folder and reference them from `doc.md` with relative paths.

### Step 4 — Doc type (governed vocabulary)
Ask for the doc type, presenting the configured `doc_types`. Validate against `doc-types.json`:
- Value in the list → accept.
- Value **not** in the list and `policy: "open"` → confirm, then append it to `doc-types.json` (transactional) and use it.
- Value **not** in the list and `policy: "locked"` → reject: "‘{type}’ isn't an approved doc type. Current types: {list}. Ask an admin to add it (`@ai:manage-doc-types`), or pick one of these." Re-prompt.
Remind, if the member offers a domain like "IT" or "Server Ops", that those are better as groups in the tree.

### Step 5 — Visibility (with the title-leak gate)
Offer the visibility choice per `library_default_visibility`.
- **Public** (default under `org_public_first`): proceed to Step 6 public path.
- **Private**: **render the blocking title-leak notice before writing anything**:
  > "The contents will be private to you and anyone you grant, but the **title `{title}` stays visible to the whole org** in the library tree. If the title itself is sensitive, give this doc a codename or a less-revealing title now."
  Offer to rename on the spot. Only after the member acknowledges/renames, proceed to Step 6 private path.

### Step 6 — Write content + meta.json
Build `meta.json` (and the matching pointer fields):
```json
{
  "id": "doc_{random}", "title": "{title}", "group_id": "{group_id}",
  "owner": "{email}", "owner_hash": "{member_hash}", "scope": "{org_public|private}",
  "type": "{type}", "tags": [...], "summary": "{1-2 sentence summary}",
  "created": "{ISO}", "updated": "{ISO}",
  "last_reviewed": "{ISO=created}", "review_cadence_days": {default}, "review_due": "{created+cadence}",
  "owner_departed": false
}
```
- **Public** → write `doc.md`, `assets/*`, and `meta.json` under `{library_path}/{group-path}/{doc-slug}/`. (For a public doc, `summary`/`tags`/`type` ARE part of `meta.json`.)
- **Private** → write the same folder under `id:{member_folder_id}/library/{doc-slug}/`. The private home is private by default (no grants needed at creation).

### Step 7 — Write the catalog pointer (same operation)
Write `{library_index_path}/{owner_hash}-{doc-slug}.json`:
- **Public** → full pointer **with** enriched fields (`type`, `tags`, `summary`) + `location` = the commons path.
- **Private** → pointer **without** `type`/`tags`/`summary`; include `id`, `title`, `group_id`, `owner`, `owner_hash`, `scope: "private"`, lifecycle dates, and `folder_id` **plus `item_drive_id`** — `aifs_stat` the doc folder once and record both its `id` (→ `folder_id`) and `drive_id` (→ `item_drive_id`, adapter 2.3.0+) — instead of a `/shared` path. `item_drive_id` lets a future grantee open the doc cross-drive (C.1.3 `crossdriveread`); it costs nothing now and avoids a re-stat at share time.
JSON-parse-verify after writing. If the pointer write fails after the content write, surface loudly and retry — do not leave the doc un-indexed.

### Step 8 — Confirm
Public: "Created `{title}` ({type}) under `{group path}` — it's org-public; anyone can read it and it'll show up in '@ai:find-doc'." Private: "Created `{title}` privately in your space. Its title shows in the tree; the contents are visible only to you until you share it with '@ai:set-doc-visibility'."

---

## Directives

- The catalog pointer and the content are one logical write — never index without content or write content without indexing.
- Enriched metadata (`type`, `tags`, `summary`) is written to the pointer **iff** the doc is org-public. Never put it in a private doc's pointer.
- Always render the title-leak warning before creating a private doc; offer the codename rename.
- A doc folder always contains `doc.md` and `meta.json`; images go in `assets/`.
- Validate `type` against `doc-types.json` per policy; in `open` mode the new type is registered org-wide.
- Use `aifs_*` for all remote operations; never native Read/Write. JSON-parse-verify every catalog write.

---

## Error Handling

- Auth/connectivity failure → halt with `@ai:member-bootstrap`; preserve the collected doc for retry.
- Permission error writing to `{library_path}` (NOT auth) → "You don't appear to have write access to the library commons — ask your admin to run `@ai:install-collection library` to provision member write access." Preserve and retry.
- Member abandons mid-interview → write nothing.
- Pointer write fails after content write → surface the inconsistency explicitly and retry the pointer write; if it can't be completed, tell the member the doc exists but isn't yet discoverable and how to retry.
