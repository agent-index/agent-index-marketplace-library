---
name: rename-item-setup
type: setup
version: 1.0.0
collection: library
description: Setup for the rename-item capability — minimal member-level setup; all configuration is org-mandated at collection install time.
target: rename-item
target_type: task
upgrade_compatible: true
---

## Setup Overview

Rename requires no member-specific configuration. All parameters are org-mandated and set during collection setup. This setup validates that the collection is installed and the member can reach the shared filesystem.

---

## Pre-Setup Checks

- Collection setup has been completed (verify `collection-setup-responses.md` exists via `aifs_read`) → if not: "Your org admin needs to complete the library collection setup first."
- Remote filesystem is accessible (`aifs_auth_status()`) → if not: "Please check your remote filesystem connection or run '@ai:member-bootstrap'."

---

## Parameters

No member-defined parameters. All configuration is inherited from the collection setup.

---

## Setup Completion

1. Validate remote filesystem access.
2. Verify the catalog exists at the configured `library_index_path` via `aifs_read`.
3. Register entry in `member-index.json` with alias `@ai:rename-item`.
4. Confirm to member: "Library is ready — say '@ai:rename-item' or just ask in plain language."

---

## Upgrade Behavior

### Preserved Responses
- None (no member-specific parameters).

### Reset on Upgrade
- None.

### Requires Member Attention
- None — org-level changes are picked up at runtime.

### Migration Notes
- v1.0.0 is the initial release.
