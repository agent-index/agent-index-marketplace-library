# Changelog â€” Library

## [1.7.1] — 2026-10-13 — public docs carry their folder id

### Fixed
- **Public doc pointers carry `folder_id` (`publicptrid`, Dev Two live test finding 49).** create-doc 1.2.2, set-doc-visibility 1.2.1 and move-item 1.0.1 record the commons doc folder's Drive id next to the `location` path; find-doc 1.4.2 opens a public doc by `id:{folder_id}` and falls back to `id:{resource_ids.shared_root}/...` when the path doesn't resolve; find-doc's and review-report 1.2.1's ACL-drift checks skip public docs (their `folder_id` is the commons). Members on Google Drive often can't resolve `/shared/...` by path; the other tasks that open public docs by path follow the core rule (CLAUDE.md template, core 4.0.0): re-read a failed `/shared/...` path as `id:{resource_ids.shared_root}/...`.

### License
- Proprietary, as before: `LICENSE` added and the `collection.json` `license` field set to `proprietary` (it said `open`).

## [1.7.0] — 2026-10-02 — Sharing and ownership transfer through the core permission-changes procedure

**MINOR — requires agent-index-core 3.32.0.** Part of retiring the permission-change-helper (design record: `/shared/projects/core-improvements/artifacts/retire-permission-helper-*.md`).

### Changed
- **`set-doc-visibility` 1.2.0** — private-doc grants and removals go through core `internal/permission-changes.md`. The owner sees the exact table and confirms it (removals one at a time), and each change is verified. After an unshare, the pointer's `scope` is derived from a fresh `aifs_get_permissions` read, so a declined or failed removal is reflected accurately.
- **Recipients outside the org are allowed** (by email; flagged **outside your org** in the table). Previously such people were dropped.
- **`transfer-doc-ownership` 1.2.0** — the current owner confirms the transfer in chat. The new owner accepts Google Drive's transfer email; until they do, the row is `awaiting_recipient` and the catalog is untouched. Re-running after they accept completes the catalog update (the procedure sees the transfer already in effect).
- **`find-doc` 1.4.1** — the owner check no longer cites the helper's Drive-API check. On gdrive adapter 2.11.1+, `aifs_get_permissions` reports a My-Drive owner as role `owner`.
- **README** lists `transfer-doc-ownership` (capability count 14).

### Fixed
- **`set-doc-visibility`** — an unshare sets `scope: "revoked"` (and says "Access revoked") only when a fresh `aifs_get_permissions` read shows no explicit grants other than the owner; otherwise it lists who still has access. Share confirmations name only satisfied rows. Removing leftover grants after going public goes through the procedure (`source: "current_state"`, each removal confirmed).
- **`set-doc-visibility`** — going private no longer deletes the commons source (non-admin members can't delete on the Shared Drive): each source file is overwritten as a moved stub and verified (soft-delete convention). Going public still removes the owner's My Drive source.
- **`transfer-doc-ownership`** — refuses `org_public` docs up front (Shared-Drive items are owned by the drive; per-item transfer is unsupported) and states that the new owner must be a registry member (the catalog needs their `owner_hash`).
- **`find-doc`** — pointer/ACL drift is flagged, never rewritten, consistent with `writes_to: null`.
- **Collection setup 1.0.1** — wording: the collaborative grants are applied by `install-collection` Step 5.5 through the core permission-changes procedure. README: private docs are readable by the owner and "people they explicitly grant".
- Manifest `external_dependencies` synced to frontmatter (`set-doc-visibility`, `transfer-doc-ownership`, `request-access`); quoted frontmatter `description` values that were invalid YAML (`archive-doc`, `library-tutorial`, `manage-doc-types`; also the `request-access` dependency entry); added `upgrade/`.

### Removed
- Dependency on the `permission-change-helper` skill and binary (frontmatter, manifests, `collection.json` external dependency, setup prerequisite).

## 1.6.1 — 2026-07-23 — Release C.1.5.2: torn-write tail reconstruction (tornwritefiledamage)

### Fixed
- **`set-doc-visibility.md` — completed the file-level relocation procedure (`tornwritefiledamage`).** The `(1) create the destination folder…` bullet was born truncated at b817e21 (no git donor ever held the rest); reconstructed `(2)` copy each file, `(3)` verify, `(4)` delete source only after the destination is confirmed — consistent with the adjacent "verify destination present, source removed" constraint. Reviewed + approved.

## 1.6.0 â€” 2026-07-18 â€” Release C.1.5.0: permission specs via the committed builder

### Changed
- **`set-doc-visibility`, `transfer-doc-ownership` â€” permission specs built by core's committed `build-permission-spec` CLI** (level-3), not hand-authored JSON. Structurally enforces the op vocabulary (belt-and-suspenders for `transferdocopname`), email/UPN recipient form, and the canonical `outputs/` spec path.

## 1.5.0 â€” 2026-07-12 â€” Release C.1.4.3: ownership op name + owner-aware reconcile

### Fixed
- **`transfer-doc-ownership` (1.1.0) â€” correct helper op name (`transferdocopname`).** The permission-change spec now names the op `transfer_ownership` (what the helper binary expects), not `transfer`.
- **`find-doc` (1.4.0) â€” owner-aware ACL reconcile (`getpermsownerwriter`).** On Google Drive, `aifs_get_permissions` reports a My-Drive item's owner as `writer` (no literal `owner` role); the reconcile no longer flags a pointer stale or heals its `owner` field merely because no `owner` role appears â€” for a My-Drive doc it trusts the pointer's `owner` and verifies via the helper's Drive-API check when a definitive owner is needed. Genuine scope/visibility drift still reconciles as before.

## 1.4.0 â€” 2026-07-10 â€” Release C.1.4.2: private-space halt remedy (memberprivspacenotprovisioned)

### Fixed
- **`create-doc` (1.2.1) â€” correct halt when the member's private space isn't provisioned.** A private doc lives in the member's OWN My Drive (`id:{member_folder_id}/library/â€¦`). If `member_folder_id` is missing, create-doc now halts with "run `@ai:member-bootstrap` to provision your `Agent-Index-Private` My Drive folder, then retry" â€” never `@ai:update` (which can't create a My Drive folder), and never a shared-drive/hashed folder. Pairs with core 3.24.0's member-bootstrap provisioning fix.

## 1.3.0 â€” 2026-07-08 â€” Release C.1.4.0: ownership transfer + pointer identity (libownerxfer)

Closes bug `20260707-8d20ea22-libownerxfer`, surfaced by the Agent Index Dev 2 gdrive-arm validation (a member ownership transfer left a stale pointer AND an undeletable duplicate the member couldn't remove).

### Added
- **`transfer-doc-ownership` (1.0.0) â€” new capability.** Transfers a doc's Drive ownership AND rewrites the catalog pointer + `meta.json` in the same operation, hard-gated on an independent `aifs_get_permissions` check; the new owner Accepts. "Make X the owner of doc Y" routes here, never through the raw permission helper (which knows nothing about the catalog). gdrive-only positive path; returns cleanly on OneDrive/SharePoint (ownership is drive/site-level â€” `NOT_IMPLEMENTED`).

### Fixed
- **Catalog pointers are keyed on the doc's stable `id`, not `owner_hash` (`create-doc` 1.2.0).** An ownership/visibility change is now an in-place content rewrite at the same filename â€” never a rename that strands an **undeletable duplicate** on the Shared-Drive index (Shared-Drive contributors can't trash files). Pointers are located by matching `id`/`folder_id`, never by reconstructing an owner-derived name; legacy `{owner_hash}-{slug}.json` files are still read by content.
- **`find-doc` (1.3.0) collapses duplicate pointers by `id`; `review-report` (1.2.0) sweeps for duplicate + legacy-named pointers** so an admin (the only role that can delete index files) can clean them up.

## 1.2.0 â€” 2026-07-06 â€” Release C.1.3.7: catalog/ACL reconcile

Closes bug `20260706-8d20ea22-libptrstale`, surfaced by the Agent Index Dev 1 gdrive-arm validation (after a raw unshare + `transferOwnership`, the `gd-test` pointer still claimed the old owner and `private_shared`).

### Fixed
- **`find-doc` (1.1.0 â†’ 1.2.0): reconcile the pointer against the actual ACL on read.** A pointer's `owner`/`scope` can drift when a doc's ownership/sharing changes outside the library (raw `transferOwnership`, out-of-band unshare, `owner_departed`). When opening a candidate, `find-doc` now does a best-effort `aifs_get_permissions` on the `folder_id` and, on mismatch, **trusts the ACL over the pointer** â€” surfaces the corrected owner/scope, notes the entry is stale, heals the pointer if writable, else flags it. Never presents a stale pointer's owner/scope as fact.
- **`review-report` (1.0.0 â†’ 1.1.0): pointer/ACL drift sweep.** Alongside overdue docs, it now flags a "stale catalog metadata" list â€” in-scope docs whose real owner/scope no longer matches the pointer â€” as the periodic reconciliation sweep `find-doc` defers to. Read-only.

## 1.1.0 â€” 2026-06-28 â€” Release C.1.3: crossdriveread

### Fixed
- **`crossdriveread` â€” shared private docs are now openable cross-drive.** A private/shared doc lives in the owner's personal OneDrive; on OneDrive a bare `id:{folder_id}` anchor resolves against the *reader's* own drive and returns `PATH_NOT_FOUND`, so a recipient could discover handoff-test-2 via its pointer but not open it (ms_prod_9). `create-doc` (1.1.0) and `set-doc-visibility` (1.1.0) now capture the doc folder's `drive_id` from `aifs_stat` and write it into the pointer (and meta) as `item_drive_id`; `find-doc` (1.1.0) opens shared private docs via the cross-drive anchor `id:{item_drive_id}:{folder_id}/doc.md` (bare-anchor fallback for older pointers), and never falls back to an external connector on a 404. Requires onedrive adapter 2.3.0+; harmless on gdrive (native shared-content reads).

## 1.0.1 â€” 2026-06-23

Patch: execution details in `set-doc-visibility` confirmed against the live gdrive adapter during Phase L. No behavior/model change.

- **Cross-drive relocation is file-level.** `aifs_copy` copies a single file and will not create the destination parent or move a whole folder in one call. make-private / make-public now spell out the procedure: create the destination folder, copy `doc.md` + `meta.json` + `assets/*` individually, verify, then delete the source.
- **Grants require a stat'd Drive ID.** `permission-change-helper` rejects a My-Drive *path* resource; share/unshare must `aifs_stat` the doc folder first and pass the bare `id:{folder_id}`.

## 1.0.0 â€” 2026-06-22

Initial release. New collection: a hierarchical documentation library replacing Confluence.

- **Taxonomy:** named groups with unlimited-depth sub-groups; always org-public. Authoritative tree in `/shared/library-index/tree.json`.
- **Documents:** folder-based (`doc.md` + `meta.json` + `assets/`) so they can carry images/attachments. Org-public by default in the `/shared/library/` commons (uniform write, changelog-attributed) or private in the owner's My Drive.
- **Privacy:** private = owner + explicit member grants (member-applied via permission-change-helper, owner-Accept, hard-gated; no owner-bypass). Private doc *titles* remain org-public â€” every make-private path surfaces a blocking title-leak warning with inline codename/rename.
- **Catalog:** always-public per-doc pointers decoupled from content ACLs; enriched retrieval metadata (`type`, `tags`, `summary`) present for public docs only and stripped on going private / re-added on going public.
- **Doc types:** governed vocabulary in `doc-types.json`, starter set (HowTo, FAQ, Tutorial, Research, Runbook, Reference), admin `locked`/`open` policy; members may add types when open. Default ships `locked`.
- **Staleness:** owner + `last_reviewed` + `review_cadence_days` (org default 180) on every doc; `review-report` lists overdue docs and pairs with a scheduled task.
- **Access requests:** routed through the org's configured comms channel.
- **Capabilities (13):** create-group, create-doc, edit-doc, move-item, rename-item, set-doc-visibility, browse-library, find-doc, request-access, archive-doc, review-report, manage-doc-types, library-tutorial.

Requires agent-index-core 3.9.0+, adapter 2.5.0+, permission-change-helper 0.4.1+.



