---
name: set-doc-visibility
type: task
version: 1.2.1
collection: library
description: Change a document's visibility and grants — make a public doc private, publish a private doc, or share/unshare a private doc with named members. Owns the title-leak warning (going private) and the strip/enrich rule (content-derived metadata exists in the catalog only while a doc is public). Private grants are applied by the owner through the core permission-changes procedure (owner confirms the exact changes; each verified; gated); no owner-bypass.
stateful: true
produces_artifacts: false
produces_shared_artifacts: true
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access (adapter 2.5.0+)
reads_from: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
writes_to: "/shared/library/, id:{member_folder_id}/library/, /shared/library-index/"
---

## About This Task

Set Document Visibility is the heart of the library's access model. It moves a document between **org-public** (the `/shared/library/` commons, readable by all) and **private** (the owner's My Drive, readable by owner + an explicit grant list), and manages the grant list for private docs. It owns two invariants:

- **Title-leak gate (decision 5):** any path that results in a private doc renders a blocking warning that the **title stays org-public**, and offers an inline codename/rename.
- **Strip/enrich rule (decision 6):** content-derived metadata (`type`, `tags`, `summary`) exists in the catalog pointer **iff** the doc is org-public. Going private **strips** those fields; going public **re-populates** them.

Private grants are **member-applied** by the owner through the core permission-changes procedure (additive `share` on the doc folder's bare `id:`): the owner sees the exact table and **confirms** it, and each grant is verified on the backend. They are **gated** (row state `applied` or `skipped_noop`) before the pointer's `scope` is updated. **No owner-bypass** — even a co-owner must be explicitly granted. Because the private home is private by default, going private is done by **relocating** the folder out of the commons (not by trying to revoke `all@` in place) — this sidesteps the fragile restriction primitive entirely (granular-access precedent).

### Inputs
The doc, the move (make-private / make-public / share-with / unshare), and for shares the people + level (reader | collaborator).

---

## Configuration
`library_path`, `library_index_path` (from setup responses). Requires agent-index-core 3.32.0+ (permission-changes procedure).

---

## Workflow

### Step 1 — Load, identify, authorize
Read setup responses and local `member-index.json` (`member_hash`, `member_folder_id` — missing → `@ai:update` halt). Verify auth. Resolve the doc via its pointer (`scope`, `location`). **Owner-only** for visibility changes — if the caller isn't the owner: "Only the owner ({owner}) can change this doc's visibility." Halt.

### Step 2 — Choose the move
> "Make `{title}` private (owner + people you grant), publish it to the whole org, or — if it's already private — share/unshare it with specific people?"

### Step 3A — Make private (public → private)
1. **Title-leak gate first:** render the blocking warning (title stays org-public) and offer to rename now.
2. **Relocate** the doc folder from `{library_path}/{path}/{slug}/` to `id:{member_folder_id}/library/{slug}/` using the **file-level relocation procedure** (see Directives — `aifs_copy` is single-file and will not create the destination parent or copy a whole folder in one call). The new home is private by default — no `all@`, no restriction primitive needed. The commons source is on the Shared Drive, where non-admin members **cannot delete** (standards.md § Addressing, convention 4), so it is **not deleted**: each source file is overwritten as a **moved stub** (no doc content, no content-derived metadata). Earlier revisions can remain in the files' version history until an admin removes the stubbed folder; flag it for admin cleanup.
3. **Strip** `type`/`tags`/`summary` from `meta.json`'s pointer-facing copy and from the catalog pointer; set `scope: "private"`, replace `location` path with `folder_id` (stat the new folder). Write + JSON-parse-verify.
4. Confirm: "`{title}` is now private. Its title still shows in the tree; contents are visible only to you until you share it."

### Step 3B — Make public (private → public)
1. **Relocate** the folder from the owner's My Drive into `{library_path}/{group-path}/{slug}/` using the **file-level relocation procedure** (see Directives). Once in the commons it inherits the `all@` reader+writer — additive, structural.
2. **Enrich:** prompt the owner to confirm/supply `summary`, `tags`, and `type` (validate `type` against `doc-types.json`). Write them into `meta.json` and the pointer; set `scope: "org_public"`, set `location` to the commons path **and `folder_id` to the commons folder's Drive id** (stat the new folder; `publicptrid`, finding 49 — members on Google Drive read public docs by id), and drop `item_drive_id` (the commons is the org drive). Write + verify.
3. Any explicit grants still on the doc are now redundant (the doc is org-readable). Leave them, or, if the owner wants them gone, revoke them through `/agent-index-core/internal/permission-changes.md`: one `unshare` per grant read from a fresh `aifs_get_permissions` on the doc folder (`source: "current_state"`, `confirmer: "owner"`), the owner confirming each removal.
4. Confirm: "`{title}` is now org-public and discoverable via '@ai:find-doc'."

### Step 3C — Share with people (private doc)
1. `aifs_stat` the doc folder → record both `folder_id` (the `id`) **and `item_drive_id` (the returned `drive_id`)** in `meta.json`. **This step is mandatory:** the permission-changes procedure rejects the composite form (e.g. `id:{member_folder_id}/library/{slug}`) and requires the folder's resolved Drive ID as a bare `id:{folder_id}`. Capturing `drive_id` (adapter 2.3.0+) is what makes the doc openable cross-drive by recipients (C.1.3 `crossdriveread`) — a private doc lives in the owner's personal OneDrive, so a recipient needs `id:{item_drive_id}:{folder_id}`.
2. Collect people + level (read | collaborate).

   **Recipients.**
   - A person in `members-registry.json` → their `sharing_identity`.
   - A name that doesn't resolve → ask for their email address. Never guess.
   - An address outside the registry is allowed. The table marks it **outside your org**, and it's the owner's call, as it would be in Drive directly. Tell the owner the recipient opens it from their storage app's "Shared with me", because the library index is for org members.
3. **Apply.** Follow `/agent-index-core/internal/permission-changes.md`:
   - `calling_task: "set-doc-visibility"`, `confirmer: "owner"`
   - one `share` per person on resource **`id:{folder_id}`** (the bare Drive ID from step 1, NOT a path), `resource_label` "Doc '{title}'", `is_folder: true`, role `reader` (read) or `writer` (collaborate), `source: "member_request"`

   **Gate:** proceed only for rows `applied` or `skipped_noop`. If none is satisfied, change nothing, report what didn't take effect, and offer to re-run.
4. If at least one grant is satisfied, set the pointer `scope: "private_shared"` and **write `item_drive_id` into the pointer** alongside `folder_id`/`item_id` (the grant list itself lives in the Drive ACL — `aifs_get_permissions` is the source of truth for *who*; the pointer records only *that* it's shared, plus the cross-drive coordinates to open it). Write + verify.
5. Confirm only the satisfied rows (`applied` / `skipped_noop`): who can now read, who can now write, and that the title was already org-visible. Name any recipient flagged outside your org. List every row that didn't take effect as **not shared yet** (pending — run this again to finish); never tell someone they have access from an unsatisfied row.

### Step 3D — Unshare
1. Ask who to remove (and whether the owner wants the doc fully withdrawn) **before** building the table.
2. Follow the permission-changes procedure with one `unshare` per grantee on `id:{folder_id}` (`source: "member_request"`, or `"current_state"` for "remove everyone"; each removal confirmed individually).
3. Re-read `aifs_get_permissions(id:{folder_id})` — derive who still has access from this **fresh read**, never from the list you intended to remove. If explicit grants (other than the owner) remain → `scope: "private_shared"`; if none remain → `scope: "private"`. A declined, cancelled or failed removal leaves that grant in place, and the scope reflects it.
4. **Full withdrawal.** If the doc's title was ever org-public and the owner wants it fully withdrawn, overwrite the pointer `scope: "revoked"` (title residue stays — accepted) **only when the fresh re-read shows no explicit grants other than the owner**. If any grant remains, do not write `"revoked"`; keep the scope from step 3 and tell the owner what's left.
5. Confirm from the re-read, honestly:
   - No explicit grants remain: "Access revoked — only you can open `{title}`. The title remains visible in the index."
   - Grants remain: "These people still have access to `{title}`: {names from the re-read}. Run this again to remove them." Never say "Access revoked" in this case.

---

## Directives

- **Owner-only.** No owner-bypass — co-owners/collaborators must be explicitly granted.
- Going private = **relocate** to My Drive; never attempt an in-place `all@` revoke (don't use the restriction/`inherit:false` primitive).
- **Strip on private, enrich on public** — content-derived metadata lives in the catalog only while the doc is org-public.
- Always render the title-leak gate before a doc becomes private.
- All grants and removals go through the core permission-changes procedure: the owner **confirms** the exact changes, and the pointer scope reflects only verified rows. Never call `aifs_share`/`aifs_unshare`/`aifs_transfer_ownership` outside it, and never write the pointer scope before the procedure returns.
- Never write `scope: "revoked"` from declined or cancelled rows — only when a fresh `aifs_get_permissions` read shows no explicit grants other than the owner.
- Verify relocations (destination present and complete; source **stubbed** when going private, removed when going public) before updating `location` / `folder_id`.
- **File-level relocation procedure** (going private and going public both use it): `aifs_copy` copies ONE file and will NOT create the destination's parent folder. So to move a doc folder across the commons ↔ My-Drive boundary: (1) create the destination folder (e.g. `aifs_write` a placeholder, or write the first file); (2) `aifs_copy` each file in turn (`doc.md`, `meta.json`, `assets/*`) into that new folder; (3) verify every file landed (stat / read-back) at the destination; (4) only then retire the source -- never touch the source before the destination is confirmed complete:
  - **Going private (source in the commons, Shared Drive):** never delete — non-admin members can't (soft-delete convention, standards.md § Addressing, convention 4). **Overwrite** each source file as a moved stub: `doc.md` → a one-line "This document has moved; find it via the library index" note; `meta.json` → `{"id": "{id}", "status": "moved", "moved_at": "{ISO-8601}"}` (no title-derived or content-derived fields beyond `id`); each `assets/*` file → an empty placeholder. Then **verify the source is stubbed** (read back each file; no original content remains).
  - **Going public (source in the owner's My Drive):** the owner may delete their own space, so remove the source folder and verify it's gone.
<!-- RECONSTRUCTED 2026-07-22: (2)-(4) of the relocation procedure were lost to a torn-write at b817e21 (bug tornwritefiledamage); completed from the bullet's own aifs_copy-one-file context, pending Bill's review. -->

<!-- AIFS:FILE-END -->
