# Changelog — Library

## 1.5.0 — 2026-07-12 — Release C.1.4.3: ownership op name + owner-aware reconcile

### Fixed
- **`transfer-doc-ownership` (1.1.0) — correct helper op name (`transferdocopname`).** The permission-change spec now names the op `transfer_ownership` (what the helper binary expects), not `transfer`.
- **`find-doc` (1.4.0) — owner-aware ACL reconcile (`getpermsownerwriter`).** On Google Drive, `aifs_get_permissions` reports a My-Drive item's owner as `writer` (no literal `owner` role); the reconcile no longer flags a pointer stale or heals its `owner` field merely because no `owner` role appears — for a My-Drive doc it trusts the pointer's `owner` and verifies via the helper's Drive-API check when a definitive owner is needed. Genuine scope/visibility drift still reconciles as before.

## 1.4.0 — 2026-07-10 — Release C.1.4.2: private-space halt remedy (memberprivspacenotprovisioned)

### Fixed
- **`create-doc` (1.2.1) — correct halt when the member's private space isn't provisioned.** A private doc lives in the member's OWN My Drive (`id:{member_folder_id}/library/…`). If `member_folder_id` is missing, create-doc now halts with "run `@ai:member-bootstrap` to provision your `Agent-Index-Private` My Drive folder, then retry" — never `@ai:update` (which can't create a My Drive folder), and never a shared-drive/hashed folder. Pairs with core 3.24.0's member-bootstrap provisioning fix.

## 1.3.0 — 2026-07-08 — Release C.1.4.0: ownership transfer + pointer identity (libownerxfer)

Closes bug `20260707-8d20ea22-libownerxfer`, surfaced by the Agent Index Dev 2 gdrive-arm validation (a member ownership transfer left a stale pointer AND an undeletable duplicate the member couldn't remove).

### Added
- **`transfer-doc-ownership` (1.0.0) — new capability.** Transfers a doc's Drive ownership AND rewrites the catalog pointer + `meta.json` in the same operation, hard-gated on an independent `aifs_get_permissions` check; the new owner Accepts. "Make X the owner of doc Y" routes here, never through the raw permission helper (which knows nothing about the catalog). gdrive-only positive path; returns cleanly on OneDrive/SharePoint (ownership is drive/site-level — `NOT_IMPLEMENTED`).

### Fixed
- **Catalog pointers are keyed on the doc's stable `id`, not `owner_hash` (`create-doc` 1.2.0).** An ownership/visibility change is now an in-place content rewrite at the same filename — never a rename that strands an **undeletable duplicate** on the Shared-Drive index (Shared-Drive contributors can't trash files). Pointers are located by matching `id`/`folder_id`, never by reconstructing an owner-derived name; legacy `{owner_hash}-{slug}.json` files are still read by content.
- **`find-doc` (1.3.0) collapses duplicate pointers by `id`; `review-report` (1.2.0) sweeps for duplicate + legacy-named pointers** so an admin (the only role that can delete index files) can clean them up.

## 1.2.0 — 2026-07-06 — Release C.1.3.7: catalog/ACL reconcile

Closes bug `20260706-8d20ea22-libptrstale`, surfaced by the Agent Index Dev 1 gdrive-arm validation (after a raw unshare + `transferOwnership`, the `gd-test` pointer still claimed the old owner and `private_shared`).

### Fixed
- **`find-doc` (1.1.0 → 1.2.0): reconcile the pointer against the actual ACL on read.** A pointer's `owner`/`scope` can drift when a doc's ownership/sharing changes outside the library (raw `transferOwnership`, out-of-band unshare, `owner_departed`). When opening a candidate, `find-doc` now does a best-effort `aifs_get_permissions` on the `folder_id` and, on mismatch, **trusts the ACL over the pointer** — surfaces the corrected owner/scope, notes the entry is stale, heals the pointer if writable, else flags it. Never presents a stale pointer's owner/scope as fact.
- **`review-report` (1.0.0 → 1.1.0): pointer/ACL drift sweep.** Alongside overdue docs, it now flags a "stale catalog metadata" list — in-scope docs whose real owner/scope no longer matches the pointer — as the periodic reconciliation sweep `find-doc` defers to. Read-only.

## 1.1.0 — 2026-06-28 — Release C.1.3: crossdriveread

### Fixed
- **`crossdriveread` — shared private docs are now openable cross-drive.** A private/shared doc lives in the owner's personal OneDrive; on OneDrive a bare `id:{folder_id}` anchor resolves against the *reader's* own drive and returns `PATH_NOT_FOUND`, so a recipient could discover handoff-test-2 via its pointer but not open it (ms_prod_9). `create-doc` (1.1.0) and `set-doc-visibility` (1.1.0) now capture the doc folder's `drive_id` from `aifs_stat` and write it into the pointer (and meta) as `item_drive_id`; `find-doc` (1.1.0) opens shared private docs via the cross-drive anchor `id:{item_drive_id}:{folder_id}/doc.md` (bare-anchor fallback for older pointers), and never falls back to an external connector on a 404. Requires onedrive adapter 2.3.0+; harmless on gdrive (native shared-content reads).

## 1.0.1 — 2026-06-23

Patch: execution details in `set-doc-visibility` confirmed against the live gdrive adapter during Phase L. No behavior/model change.

- **Cross-drive relocation is file-level.** `aifs_copy` copies a single file and will not create the destination parent or move a whole folder in one call. make-private / make-public now spell out the procedure: create the destination folder, copy `doc.md` + `meta.json` + `assets/*` individually, verify, then delete the source.
- **Grants require a stat'd Drive ID.** `permission-change-helper` rejects a My-Drive *path* resource; share/unshare must `aifs_stat` the doc folder first and pass the bare `id:{folder_id}`.

## 1.0.0 — 2026-06-22

Initial release. New collection: a hierarchical documentation library replacing Confluence.

- **Taxonomy:** named groups with unlimited-depth sub-groups; always org-public. Authoritative tree in `/shared/library-index/tree.json`.
- **Documents:** folder-based (`doc.md` + `meta.json` + `assets/`) so they can carry images/attachments. Org-public by default in the `/shared/library/` commons (uniform write, changelog-attributed) or private in the owner's My Drive.
- **Privacy:** private = owner + explicit member grants (member-applied via permission-change-helper, owner-Accept, hard-gated; no owner-bypass). Private doc *titles* remain org-public — every make-private path surfaces a blocking title-leak warning with inline codename/rename.
- **Catalog:** always-public per-doc pointers decoupled from content ACLs; enriched retrieval metadata (`type`, `tags`, `summary`) present for public docs only and stripped on going private / re-added on going public.
- **Doc types:** governed vocabulary in `doc-types.json`, starter set (HowTo, FAQ, Tutorial, Research, Runbook, Reference), admin `locked`/`open` policy; members may add types when open. Default ships `locked`.
- **Staleness:** owner + `last_reviewed` + `review_cadence_days` (org default 180) on every doc; `review-report` lists overdue docs and pairs with a scheduled task.
- **Access requests:** routed through the org's configured comms channel.
- **Capabilities (13):** create-group, create-doc, edit-doc, move-item, rename-item, set-doc-visibility, browse-library, find-doc, request-access, archive-doc, review-report, manage-doc-types, library-tutorial.

Requires agent-index-core 3.9.0+, adapter 2.5.0+, permission-change-helper 0.4.1+.
