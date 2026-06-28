# Changelog — Library

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
