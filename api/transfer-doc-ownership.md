---
name: transfer-doc-ownership
type: task
version: 1.2.1
collection: library
description: Transfer ownership of a library document to another member, performing the Drive ownership change AND the catalog bookkeeping atomically. Routes "make X the owner of doc Y" here — never as a bare ownership change (which knows nothing about the catalog and leaves the pointer stale). The pointer is rewritten in place (it is keyed on the doc's stable id, so ownership transfer is never a rename), so it never creates an undeletable duplicate. gdrive only — ownership transfer is NOT_IMPLEMENTED on OneDrive/SharePoint (ownership is drive/site-level).
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

Transfer Document Ownership moves a library doc's ownership from the current owner to another member and keeps the catalog consistent in the **same operation**. It exists to close bug `libownerxfer`: before it, "make Bill the owner" went through the raw permission helper (a Drive-only ownership change that knows nothing about the catalog), which left the pointer + `meta.json` recording the old owner (`libptrstale` drift) and — when anyone tried to fix the owner-hash-keyed pointer filename by renaming it — created a **duplicate pointer the member couldn't delete** (Shared-Drive contributors can't trash index files).

Two things make this safe now:

- **The pointer is keyed on the doc's stable `id`, not on `owner_hash`** (create-doc 1.1.0+). So an ownership change is an **in-place content rewrite at the same filename** — never a rename, never a create-new-plus-delete-old, never a duplicate.
- **The Drive transfer and the catalog rewrite happen together and are hard-gated** — the pointer is not touched until the ACL change is independently confirmed.

### Inputs
The doc, and the new owner (a member, resolved against `members-registry.json`).

---

## Configuration
`library_path`, `library_index_path` (from setup responses). Requires agent-index-core 3.32.0+ (permission-changes procedure).

---

## Workflow

### Step 1 — Load, identify, authorize
Read setup responses and local `member-index.json` (`member_hash`, `member_folder_id` — missing → `@ai:update` halt). Verify auth. Resolve the doc via its pointer (locate it by matching the doc's `id`/`folder_id`, **not** by reconstructing a filename). **Owner-only:** only the current owner may transfer ownership — if the caller isn't the pointer's `owner`, halt: "Only the current owner ({owner}) can transfer this doc." On gdrive adapter 2.11.1+ the caller's ownership can also be confirmed via `aifs_get_permissions` on the doc folder (role `owner`); on older adapters the owner shows as `writer`, so trust the pointer. Resolve the new owner against `members-registry.json`. **The new owner must be a registry member** — unlike sharing, an outside email isn't accepted, because the catalog pointer and `meta.json` must record the new owner's `owner_hash`, which only registry members have. If the name doesn't resolve, ask for the member's email; if it still isn't in the registry, halt: "{name} isn't a member of the org, so they can't own a library doc. Share it with them instead (`@ai:set-doc-visibility`)."

### Step 2 — Scope and backend gates
**Scope pre-check (first).** If the pointer's `scope` is `org_public`, the doc lives in the Shared Drive commons, where the shared drive owns every item and per-item ownership transfer is unsupported. Halt before any permission change: "Public docs are owned by the shared drive, so their ownership can't move. To change the owner, make `{title}` private first (`@ai:set-doc-visibility`) and then transfer it, or ask an admin." Don't build a transfer row for it.

**Backend gate.** Ownership transfer is a **gdrive-only positive path**. If the org's backend is OneDrive/SharePoint, halt cleanly: "Ownership transfer isn't supported on this backend — OneDrive/SharePoint ownership is drive/site-level, not per-item. Consider sharing the doc as a collaborator instead (`@ai:set-doc-visibility`)." This mirrors the adapter's `NOT_IMPLEMENTED` for `transferOwnership` and is the *correct* pass on onedrive, not a failure.

### Step 3 — Transfer Drive ownership (hard-gated)
1. `aifs_stat` the doc folder → confirm its `id` (`folder_id`) and current `drive_id` (`item_drive_id`).
2. **Request the transfer.** Follow `/agent-index-core/internal/permission-changes.md`:
   - `calling_task: "transfer-doc-ownership"`, `confirmer: "owner"`
   - one change: `op: "transfer_ownership"` on resource **`id:{folder_id}`** (the bare Drive ID, NOT a path) to the new owner's `sharing_identity`, `resource_label` "Doc '{title}'", `source: "member_request"`

   The **current owner confirms** in chat (an ownership transfer is always confirmed individually). Before they confirm, tell them where the doc will live (finding 50): "The doc folder stays where it is, inside your private space — Drive moves the ownership, not the folder. {new_owner} will find it under 'Shared with me' in Drive, or through the library. You'll keep edit access through your space unless you remove it afterwards." Google then emails the **new owner, who must accept** in Google Drive; that consent happens outside agent-index. The transfer covers the doc folder and its contents (`doc.md`, `meta.json`, `assets/*`) as the adapter's transfer covers.
3. **Gate on the row state.** Never write the catalog before it passes.
   - **`applied`** (the backend shows the new owner) or **`skipped_noop`** (the new owner already owns it, e.g. a re-run after they accepted) → Step 4.
   - **`awaiting_recipient`** → the new owner hasn't accepted yet. Do NOT touch the catalog (it still correctly records the current owner). Tell the owner: "I've asked Google Drive to transfer `{title}` to {new_owner}. They'll get an email to accept. Once they do, run this again and I'll update the library catalog." Stop.
   - Anything else → report it with the procedure's summary line; write nothing.

### Step 4 — Catalog bookkeeping (in place, atomic with the transfer)
Only after the gate passes:
1. **Re-stat** the doc folder — the folder stays parented in the previous owner's space, but its `drive_id` can still change with the owner. Capture the fresh `id` (`folder_id`) and `drive_id` (`item_drive_id`).
2. **Rewrite the pointer in place at its existing path** (found in Step 1 by `id`): set `owner` = new owner email, `owner_hash` = new owner hash, refresh `item_drive_id`/`folder_id`, bump `updated`. Do **not** change `id`, `title`, `scope`, `group_id`, or the filename. JSON-parse-verify.
   - **Legacy pointer (old `{owner_hash}-{slug}.json` filename):** still rewrite the content **in place at that filename** — do NOT rename it (a rename needs a delete the member can't do). Note that the filename is cosmetically stale and flag it for admin migration via `@ai:review-report`. Never create a second pointer.
3. **Update `meta.json`** owner/owner_hash to match (via `aifs_write` at the resolved doc location). Verify.

### Step 5 — Confirm
"Ownership of `{title}` transferred to {new_owner}. The catalog and doc metadata now record {new_owner} as owner." Then explain where it lives, for both people (finding 50): "The doc folder is still inside {previous_owner}'s private space — Drive moved the ownership, not the folder. {new_owner}: you'll find it under 'Shared with me' in Drive, or through the library ('find {title}'). {previous_owner}: you still have edit access through your space; remove it from your space if you don't want that." If a legacy filename was left in place, add: "(An admin can tidy the catalog filename with `@ai:review-report`; the entry itself is correct.)"

---

## Directives

- **Owner-only**: the current owner confirms the transfer in chat, and the new owner must accept it in Google Drive. The catalog is **gated** on the verified transfer (`applied` / `skipped_noop`) before any pointer/meta write.
- **Never rename the pointer** — it is keyed on the stable doc `id`; ownership is a content-field change. Never create a second pointer for the same `id`.
- Route "make X the owner of doc Y" here; never make the ownership change outside this task (that's what strands the catalog — `libownerxfer`/`libptrstale`). Never call `aifs_transfer_ownership` outside the core permission-changes procedure.
- Use `aifs_*` for all remote ops; JSON-parse-verify every catalog write.
- gdrive-only; onedrive returns cleanly at Step 2. `org_public` docs (Shared-Drive-owned) also stop at Step 2 — never attempt a transfer on them.
- The new owner must be a `members-registry.json` member (the catalog needs their `owner_hash`).
- **The folder doesn't move (finding 50).** Drive keeps the doc folder parented in the previous owner's private space after a transfer, so the previous owner keeps writer access by inheritance and the new owner sees the doc under "Shared with me". This task leaves it there and says so in the Step 3 confirmation and the Step 5 result. Never move the folder as part of a transfer.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`; write nothing.
- New owner hasn't accepted (`awaiting_recipient`) or declines in Drive → the transfer never completes; do NOT touch the catalog (it still correctly records the old owner). Surface that the transfer is pending the recipient's acceptance; a re-run after they accept completes the catalog update (the procedure sees the transfer already in effect).
- Transfer applied but a catalog write fails → the ACL is authoritative; surface loudly that ownership moved but the catalog write must be retried, and retry the in-place pointer/meta write. `find-doc`'s read-reconcile will report the corrected owner from the ACL in the meantime, so the doc is never misattributed.
- Caller isn't the owner, or new owner isn't a registry member → halt in Step 1.
- Doc is `org_public` → halt in Step 2 with the "owned by the shared drive" message; write nothing.

<!-- AIFS:FILE-END -->
