---
name: set-doc-visibility
type: task
version: 1.1.0
collection: library
description: Change a document's visibility and grants — make a public doc private, publish a private doc, or share/unshare a private doc with named members. Owns the title-leak warning (going private) and the strip/enrich rule (content-derived metadata exists in the catalog only while a doc is public). Private grants are member-applied via permission-change-helper, owner-Accept, hard-gated; no owner-bypass.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills:
    - permission-change-helper
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
  - permission-change-helper binary 0.4.1+ (bare id:{folderId} resources)
reads_from: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
writes_to: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
---

## About This Task

Set Document Visibility is the heart of the library's access model. It moves a document between **org-public** (the `/shared/library/` commons, readable by all) and **private** (the owner's My Drive, readable by owner + an explicit grant list), and manages the grant list for private docs. It owns two invariants:

- **Title-leak gate (decision 5):** any path that results in a private doc renders a blocking warning that the **title stays org-public**, and offers an inline codename/rename.
- **Strip/enrich rule (decision 6):** content-derived metadata (`type`, `tags`, `summary`) exists in the catalog pointer **iff** the doc is org-public. Going private **strips** those fields; going public **re-populates** them.

Private grants are **member-applied** by the owner via `permission-change-helper` (additive `op:share` on the doc folder's bare `id:`), owner **Accepts**, and are **hard-gated** (outcome `applied` OR an independent `aifs_get_permissions`) before the pointer's `scope` is updated. **No owner-bypass** — even a co-owner must be explicitly granted. Because the private home is private by default, going private is done by **relocating** the folder out of the commons (not by trying to revoke `all@` in place) — this sidesteps the fragile restriction primitive entirely (granular-access precedent).

### Inputs
The doc, the move (make-private / make-public / share-with / unshare), and for shares the people + level (reader | collaborator).

---

## Configuration
`library_path`, `library_index_path` (from setup responses). Requires `permission-change-helper` 0.4.1+.

---

## Workflow

### Step 1 — Load, identify, authorize
Read setup responses and local `member-index.json` (`member_hash`, `member_folder_id` — missing → `@ai:update` halt). Verify auth. Resolve the doc via its pointer (`scope`, `location`). **Owner-only** for visibility changes — if the caller isn't the owner: "Only the owner ({owner}) can change this doc's visibility." Halt.

### Step 2 — Choose the move
> "Make `{title}` private (owner + people you grant), publish it to the whole org, or — if it's already private — share/unshare it with specific people?"

### Step 3A — Make private (public → private)
1. **Title-leak gate first:** render the blocking warning (title stays org-public) and offer to rename now.
2. **Relocate** the doc folder from `{library_path}/{path}/{slug}/` to `id:{member_folder_id}/library/{slug}/` using the **file-level relocation procedure** (see Directives — `aifs_copy` is single-file and will not create the destination parent or copy a whole folder in one call). The new home is private by default — no `all@`, no restriction primitive needed.
3. **Strip** `type`/`tags`/`summary` from `meta.json`'s pointer-facing copy and from the catalog pointer; set `scope: "private"`, replace `location` path with `folder_id` (stat the new folder). Write + JSON-parse-verify.
4. Confirm: "`{title}` is now private. Its title still shows in the tree; contents are visible only to you until you share it."

### Step 3B — Make public (private → public)
1. **Relocate** the folder from the owner's My Drive into `{library_path}/{group-path}/{slug}/` using the **file-level relocation procedure** (see Directives). Once in the commons it inherits the `all@` reader+writer — additive, structural.
2. **Enrich:** prompt the owner to confirm/supply `summary`, `tags`, and `type` (validate `type` against `doc-types.json`). Write them into `meta.json` and the pointer; set `scope: "org_public"`, replace `folder_id` with the commons `location` path. Write + verify.
3. Any prior private grants are now moot (the doc is org-readable) — they can be left or revoked harmlessly.
4. Confirm: "`{title}` is now org-public and discoverable via '@ai:find-doc'."

### Step 3C — Share with people (private doc)
1. `aifs_stat` the doc folder → record both `folder_id` (the `id`) **and `item_drive_id` (the returned `drive_id`)** in `meta.json`. **This step is mandatory:** the helper rejects a My-Drive *path* (e.g. `id:{member_folder_id}/library/{slug}`) and requires the folder's resolved Drive ID as a bare `id:{folder_id}`. Capturing `drive_id` (adapter 2.3.0+) is what makes the doc openable cross-drive by recipients (C.1.3 `crossdriveread`) — a private doc lives in the owner's personal OneDrive, so a recipient needs `id:{item_drive_id}:{folder_id}`.
2. Collect people + level (read | collaborate); resolve against `members-registry.json`; drop unresolvables with notice.
3. ONE `permission-change-helper` spec: `op: "share"` per grant on resource **`id:{folder_id}`** (the bare Drive ID from step 1, NOT a path), role `reader` (read) or `writer` (collaborate). **Owner Accepts.** **HARD GATE:** proceed only on outcome `"applied"` OR an independent `aifs_get_permissions` confirming each grant. **Build this spec with the committed `build-permission-spec` CLI** (see `permission-change-helper` Step 2.5): emit the ops-array as data and run the CLI -- it enforces the op name, email/UPN recipient form, required role, and the canonical `<project_dir>/outputs/` path, and prints the `spec_path`/`link_path` to use. Do not hand-author the spec JSON. (Applies to the `unshare` variant below too.)
4. After the gate passes, set the pointer `scope: "private_shared"` and **write `item_drive_id` into the pointer** alongside `folder_id`/`item_id` (the grant list itself lives in the Drive ACL — `aifs_get_permissions` is the source of truth for *who*; the pointer records only *that* it's shared, plus the cross-drive coordinates to open it). Write + verify.
5. Confirm who can read, who can write, and that the title was already org-visible.

### Step 3D — Unshare
ONE spec `op: "unshare"` per grantee on `id:{folder_id}`; owner Accepts; hard gate. If grants remain → `scope: "private_shared"`; if none remain → `scope: "private"`. If the doc's title was ever org-public and the owner wants it fully withdrawn, the pointer is overwritten `scope: "revoked"` (title residue stays — accepted). Confirm honestly: "Access revoked; the title remains visible in the index."

---

## Directives

- **Owner-only.** No owner-bypass — co-owners/collaborators must be explicitly granted.
- Going private = **relocate** to My Drive; never attempt an in-place `all@` revoke (don't use the restriction/`inherit:false` primitive).
- **Strip on private, enrich on public** — content-derived metadata lives in the catalog only while the doc is org-public.
- Always render the title-leak gate before a doc becomes private.
- All grants via `permission-change-helper`, owner **Accepts**, **hard-gated** before any pointer scope change. Never write the pointer scope before the gate passes.
- Verify relocations (destination present, source removed) before updating `location`.
- **File-level relocation procedure** (going private and going public both use it): `aifs_copy` copies ONE file and will NOT create the destination's parent folder. So to move a doc folder across the commons ↔ My-Drive boundary: (1) create the destination folder (e.g. `aifs_write` a placeholder, or write the first file); (2) `aifs_copy` each file in turn (`doc.md`, `meta.json`, `assets/*`) into that new folder; (3) verify every file landed (stat / read-back) at the destination; (4) only then remove the source folder -- never delete the source before the destination is confirmed complete.
<!-- RECONSTRUCTED 2026-07-22: (2)-(4) of the relocation procedure were lost to a torn-write at b817e21 (bug tornwritefiledamage); completed from the bullet's own aifs_copy-one-file context, pending Bill's review. -->
