---
name: library-collection-setup
type: collection-setup
version: 1.0.0
collection: library
description: Org-admin setup for the library collection — provisions the public-doc commons and the always-public catalog, seeds the taxonomy and the governed doc-type vocabulary, and configures default visibility, review cadence, and the access-request comms channel.
upgrade_compatible: true
---

## Collection Setup Overview

This setup is run by an org admin when installing the library collection. It creates the two filesystem roots the library uses — the public-document commons (`/shared/library/`) and the always-org-public catalog (`/shared/library-index/`) — seeds an empty taxonomy and the starter doc-type vocabulary, and records org defaults (default visibility, review cadence). The additive collaborative grants in `collaborative-acls.json` are provisioned by `install-collection` Step 5.5 (admin Accepts).

---

## Prerequisites

- Remote filesystem access via `aifs_*` tools (test with `aifs_auth_status()`).
- `permission-change-helper` binary 0.4.1+ available (members apply their own private-doc grants at runtime).
- The org's `org-config.json` should define the all-members group and, if access-request notifications are wanted, a comms channel (Slack / Teams / Discord / email).

---

## Parameters

### `library_path` [org-mandated]
Remote filesystem path for the public-document commons.
- Default: `/shared/library/`
- Ask: "Where should public documents be stored on the remote filesystem? Default is '/shared/library/'. Every member can read and collaboratively edit docs here; private docs live in their owner's My Drive instead."
- Setup creates this directory with the all-members reader+writer grant (via `collaborative-acls.json`).

### `library_index_path` [org-mandated]
Remote filesystem path for the always-public catalog.
- Default: `/shared/library-index/`
- Ask: "Where should the library catalog (the public taxonomy + per-doc index) live? Default is '/shared/library-index/'. This is org-public to all members and holds the tree, one pointer per doc, and the doc-type vocabulary."
- Setup creates this directory with the all-members writer grant and seeds `tree.json` and `doc-types.json`.

### `library_default_visibility` [org-mandated]
Default visibility offered when a member creates a document.
- Default: `org_public_first`
- Options: `org_public_first` (public is the default, member can choose private) · `ask` (no default; always prompt) · `private_first` (private is the default).
- Ask: "When members create a document, what should the default visibility be? Default is 'org_public_first' — public unless they choose private."

### `library_default_review_cadence_days` [org-mandated]
Default number of days after which a document is flagged for review.
- Default: `180`
- Ask: "How many days should a document stay 'fresh' before it's flagged for review? Default is 180."

### `doc_type_policy` [org-mandated]
Whether the doc-type vocabulary is locked to admin-approved values or open to member additions.
- Default: `locked`
- Options: `locked` (only admin-approved types may be used) · `open` (any member may introduce a new type at doc creation, which is then registered org-wide).
- Ask: "Should document types be locked to an admin-approved list, or open so members can add their own? Default is 'locked'."
- Stored in `{library_index_path}/doc-types.json` alongside the types list.

### `doc_types` [org-mandated]
The starter document-type vocabulary.
- Default: `["HowTo", "FAQ", "Tutorial", "Research", "Runbook", "Reference"]`
- Ask: "What document types should the library start with? Default: HowTo, FAQ, Tutorial, Research, Runbook, Reference. (Types describe FORMAT; organizational domains like 'IT' or 'Server Ops' should be groups in the tree, not types.)"

### `access_request_channel` [org-mandated]
How `request-access` notifies a private doc's owner.
- Default: read from the org's existing comms config in `org-config.json` (no new value needed if one is configured).
- Ask: "When a member requests access to a private doc, how should the owner be notified? Default is to use the org's configured comms channel ({detected from org-config.json}). If none is configured, requests fall back to surfacing the owner's name so the requester can reach out directly."

---

## Setup Completion

1. Validate remote filesystem access via `aifs_auth_status()`. If not authenticated, attempt `aifs_authenticate`. If that fails, halt and instruct the admin.
2. Create the commons directory at `{library_path}` via `aifs_write` (write a `.gitkeep` placeholder if needed to create the directory).
3. Create the catalog directory at `{library_index_path}` via `aifs_write`.
4. Seed an empty taxonomy at `{library_index_path}/tree.json`:
   ```json
   { "version": 1, "groups": [] }
   ```
5. Seed the doc-type vocabulary at `{library_index_path}/doc-types.json`:
   ```json
   { "policy": "{doc_type_policy}", "types": {doc_types} }
   ```
6. Confirm the all-members reader+writer grant on `{library_path}/` and writer grant on `{library_index_path}/` are provisioned (these are declared in `collaborative-acls.json` and applied by `install-collection` Step 5.5 — verify via `aifs_get_permissions`, not the transcript).
7. Write `collection-setup-responses.md` to `/setup/` with all configured parameters in YAML.
8. Confirm to admin: "Library is set up. Members can create groups and documents and browse with natural language ('what docs do we have', 'create a how-to under …'). Doc types are currently {doc_type_policy}. Default new-doc visibility is {library_default_visibility}; docs are flagged for review after {library_default_review_cadence_days} days."

---

## Upgrade Behavior

### Preserved Responses
- `library_path`, `library_index_path`, `library_default_visibility`, `library_default_review_cadence_days`, `doc_type_policy`, `doc_types`, `access_request_channel`
- All existing groups, documents, pointers, and the doc-type vocabulary.

### Reset on Upgrade
- None.

### Requires Member Attention
- If `doc_type_policy` is tightened from `open` to `locked`, documents already tagged with now-unlisted types keep their type until next edited; `manage-doc-types` can list affected docs.

### Migration Notes
- v1.0.0 is the initial release. Greenfield install — no data migration. Future migration notes will be added here.
