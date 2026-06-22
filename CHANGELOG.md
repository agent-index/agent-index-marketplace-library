# Changelog — Library

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
