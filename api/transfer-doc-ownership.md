---
name: transfer-doc-ownership
type: task
version: 1.1.0
collection: library
description: Transfer ownership of a library document to another member, performing the Drive ownership change AND the catalog bookkeeping atomically. Routes "make X the owner of doc Y" here — never through the raw permission helper (which knows nothing about the catalog and leaves the pointer stale). The pointer is rewritten in place (it is keyed on the doc's stable id, so ownership transfer is never a rename), so it never creates an undeletable duplicate. gdrive only — ownership transfer is NOT_IMPLEMENTED on OneDrive/SharePoint (ownership is drive/site-level).
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

Transfer Document Ownership moves a library doc's ownership from the current owner to another member and keeps the catalog consistent in the **same operation**. It exists to close bug `libownerxfer`: before it, "make Bill the owner" went through the raw `permission-change-helper` (a Drive-only ownership change that knows nothing about the catalog), which left the pointer + `meta.json` recording the old owner (`libptrstale` drift) and — when anyone tried to fix the owner-hash-keyed pointer filename by renaming it — created a **duplicate pointer the member couldn't delete** (Shared-Drive contributors can't trash index files).

Two things make this safe now:

- **The pointer is keyed on the doc's stable `id`, not on `owner_hash`** (create-doc 1.1.0+). So an ownership change is an **in-place content rewrite at the same filename** — never a rename, never a create-new-plus-delete-old, never a duplicate.
- **The Drive transfer and the catalog rewrite happen together and are hard-gated** — the pointer is not touched until the ACL change is independently confirmed.

### Inputs
The doc, and the new owner (a member, resolved against `members-registry.json`).

---

## Configuration
`library_path`, `library_index_path` (from setup responses). Requires `permission-change-helper` 0.4.1+.

---

## Workflow

### Step 1 — Load, identify, authorize
Read setup responses and local `member-index.json` (`member_hash`, `member_folder_id` — missing → `@ai:update` halt). Verify auth. Resolve the doc via its pointer (locate it by matching the doc's `id`/`folder_id`, **not** by reconstructing a filename). **Owner-only:** only the current owner may transfer ownership — if the caller isn't the pointer's `owner`, halt: "Only the current owner ({owner}) can transfer this doc." Resolve the new owner against `members-registry.json`; if unresolvable, halt with a clear message.

### Step 2 — Backend capability gate
Ownership transfer is a **gdrive-only positive path**. If the org's backend is OneDrive/SharePoint, halt cleanly: "Ownership transfer isn't supported on this backend — OneDrive/SharePoint ownership is drive/site-level, not per-item. Consider sharing the doc as a collaborator instead (`@ai:set-doc-visibility`)." This mirrors the adapter's `NOT_IMPLEMENTED` for `transferOwnership` and is the *correct* pass on onedrive, not a failure.

### Step 3 — Transfer Drive ownership (hard-gated)
1. `aifs_stat` the doc folder → confirm its `id` (`folder_id`) and current `drive_id` (`item_drive_id`).
2. ONE `permission-change-helper` spec: `op: "transfer_ownership"` (transferOwnership) on resource **`id:{folder_id}`** (the bare Drive ID, NOT a path) to the new owner. The **new owner Accepts** the ownership transfer (Google requires the recipient's consent). Apply to the doc folder and its contents (`doc.md`, `meta.json`, `assets/*`) as the adapter's transfer covers.
3. **HARD GATE:** proceed only on outcome `"applied"` OR an independent `aifs_get_permissions` on `id:{folder_id}` confirming the new owner is now `owner` (and the prior owner is demoted). Never write the catalog before this passes.

### Step 4 — Catalog bookkeeping (in place, atomic with the transfer)
Only after the gate passes:
1. **Re-stat** the doc folder — a transferred private doc may now live in the new owner's My Drive, so its `drive_id` can change. Capture the fresh `id` (`folder_id`) and `drive_id` (`item_drive_id`).
2. **Rewrite the pointer in place at its existing path** (found in Step 1 by `id`): set `owner` = new owner email, `owner_hash` = new owner hash, refresh `item_drive_id`/`folder_id`, bump `updated`. Do **not** change `id`, `title`, `scope`, `group_id`, or the filename. JSON-parse-verify.
   - **Legacy pointer (old `{owner_hash}-{slug}.json` filename):** still rewrite the content **in place at that filename** — do NOT rename it (a rename needs a delete the member can't do). Note that the filename is cosmetically stale and flag it for admin migration via `@ai:review-report`. Never create a second pointer.
3. **Update `meta.json`** owner/owner_hash to match (via `aifs_write` at the resolved doc location). Verify.

### Step 5 — Confirm
"Ownership of `{title}` transferred to {new_owner}. The catalog and doc metadata now record {new_owner} as owner." If a legacy filename was left in place, add: "(An admin can tidy the catalog filename with `@ai:review-report`; the entry itself is correct.)"

---

## Directives

- **Owner-only**, new owner must **Accept**, and the catalog is **hard-gated** on an independent ACL confirmation before any pointer/meta write.
- **Never rename the pointer** — it is keyed on the stable doc `id`; ownership is a content-field change. Never create a second pointer for the same `id`.
- Route "make X the owner of doc Y" here; never send an ownership change through the raw `permission-change-helper` directly (that's what strands the catalog — `libownerxfer`/`libptrstale`).
- Use `aifs_*` for all remote ops; JSON-parse-verify every catalog write.
- gdrive-only; onedrive returns cleanly at Step 2.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`; write nothing.
- New owner declines / doesn't Accept → the transfer never completes; do NOT touch the catalog (it still correctly records the old owner). Surface that the transfer is pending the recipient's acceptance.
- Transfer applied but a catalog write fails → the ACL is authoritative; surface loudly that ownership moved but the catalog write must be retried, and retry the in-place pointer/meta write. `find-doc`'s read-reconcile will report the corrected owner from the ACL in the meantime, so the doc is never misattributed.
- Caller isn't the owner, or new owner unresolvable → halt in Step 1.

<!-- AIFS:FILE-END -->
