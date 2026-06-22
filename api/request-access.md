---
name: request-access
type: task
version: 1.0.0
collection: library
description: Ask a private document's owner for read access. Sends a notification to the owner via the org's configured comms channel (Slack/Teams/Discord/email) with the doc title, id, and requester. The grant itself happens when the owner runs set-doc-visibility; this only delivers the request.
stateful: false
produces_artifacts: false
produces_shared_artifacts: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
  - Slack / Microsoft Teams / Discord / email (conditional): deliver the access-request notification via the org's configured channel
reads_from: "/shared/library-index/"
writes_to: null
---

## About This Task

Request Access lets a member ask the owner of a **private** document for read access. The member can see private doc titles in the tree (the taxonomy is org-public) but not their contents. This task delivers a request to the owner via the org's **configured comms channel** (Slack / Teams / Discord / email — read from `org-config.json`). It does **not** grant anything itself — the grant happens when the owner runs `@ai:set-doc-visibility` → share-with. If no comms channel is configured, it falls back to surfacing the owner's name so the requester can reach out directly.

---

## Configuration
`library_index_path`, and `access_request_channel` (resolved from the org's comms config) — from setup responses.

---

## Workflow

### Step 1 — Load and resolve
Read setup responses for `library_index_path` and the configured comms channel. Read local `member-index.json` for the requester's identity. Verify auth (`aifs_auth_status`). Resolve the target doc via its catalog pointer with `aifs_read("{library_index_path}/{owner_hash}-{doc-slug}.json")` (locate by title via the tree/pointers if the slug isn't known) → `title`, `id`, `owner`, `owner_hash`, `scope`. Resolve the owner's contact identity from `members-registry.json` via `aifs_read`. Use `aifs_*` for all remote reads — never native Read.

### Step 2 — Sanity checks
- If the doc is `org_public` → "That doc is already org-public — you can read it now (`@ai:find-doc`)." No request needed.
- If the requester is already the owner or already granted → say so; no request needed.
- If the doc is private/`private_shared` and the requester isn't granted → proceed.

### Step 3 — Compose the request
Ask the requester for an optional note ("why you need it"). Build the message:
> "{requester display_name} is requesting read access to the private library doc **{title}** (id `{id}`). {optional note}. Grant it with `@ai:set-doc-visibility` → share with {requester}."

### Step 4 — Deliver via the configured channel
Send to the **owner** over the org's configured comms channel (DM on Slack/Teams/Discord, or email), using the org's existing comms integration. If no channel is configured, fall back: "I can't send this automatically — your org hasn't configured a comms channel. The owner is {owner display_name} ({email}); reach out to them and ask them to share `{title}` with you."

### Step 5 — Confirm
"Sent your access request for `{title}` to {owner}. They grant access from their side; you'll be able to read it once they do."

---

## Directives

- This task never grants access and never writes to the catalog or doc — it only delivers a message.
- Route via the org's configured comms channel; degrade gracefully (surface the owner) if none exists.
- Never reveal or infer the private doc's contents in the request flow.
- Resolve identities against `members-registry.json`.

---

## Error Handling

- Auth/connectivity (filesystem) → halt + `@ai:member-bootstrap`.
- Comms-channel delivery fails → fall back to surfacing the owner's name/email so the requester can reach out; report that the automatic send didn't go through.
- Doc not found in the catalog → "I can't find a doc by that title — try `@ai:browse-library` to locate it."
