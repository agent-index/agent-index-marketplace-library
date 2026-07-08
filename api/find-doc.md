---
name: find-doc
type: skill
version: 1.3.0
collection: library
description: The primary read path. Query the catalog (title, tags, summary, type) for docs on a topic, then open only the matching docs the member is allowed to read and summarize them. Private docs the member can't read surface title-only with a request-access affordance.
stateful: false
dependencies:
  skills: []
  tasks: []
external_dependencies:
  - Remote filesystem access
reads_from: "/shared/library-index/, /shared/library/, id:{member_folder_id}/library/"
writes_to: null
---

## About This Skill

Find Document is the **primary read path**. It answers "what do we have on X" / "how do I Y" by querying the catalog's metadata (title, tags, summary, type — enriched fields exist for public docs; private docs match on title only), ranking the candidates, then opening **only** the matching docs the member is allowed to read and summarizing them. It never opens docs the member can't read; those surface title-only with a request-access affordance.

This metadata-first approach is what lets the library scale past a few hundred docs — Claude routes to 2–3 candidates instead of reading everything.

---

## Configuration
`library_index_path` (from setup responses).

---

## Workflow

### Step 1 — Load
Read setup responses; verify auth. Read `tree.json` and the per-doc pointers.

**Collapse duplicate pointers by doc `id` (`libownerxfer`).** Index pointers by their `id` field, not by filename. If two or more pointers share the same `id` — a legacy artifact of the old `{owner_hash}-{slug}` naming, where an ownership change created a second file the member couldn't delete — treat them as ONE doc: keep the entry whose `owner`/`scope` matches the actual ACL (Step 3 reconcile) and suppress the stale twin, noting "`{title}` has a duplicate catalog pointer — an admin should remove the stale one (`@ai:review-report` lists it)." Never present the same doc twice.

### Step 2 — Match
Score pointers against the member's query over `title`, `tags`, `summary`, and `type` (public docs) and `title` (private docs). Rank; keep the top few candidates.

### Step 3 — Open only what's readable
For each top candidate, resolve the read target from its pointer:
- **`org_public`** → read `doc.md` at the commons `location` path.
- **`private`/`private_shared`** → the doc lives in the **owner's** personal drive. Build the **cross-drive anchor** `id:{item_drive_id}:{folder_id}/doc.md` when the pointer carries `item_drive_id` (C.1.3 `crossdriveread`); fall back to the bare `id:{folder_id}/doc.md` only for older pointers that predate `item_drive_id`. A bare anchor for a doc on another member's drive resolves against *this* member's drive and returns `PATH_NOT_FOUND` even when the grant exists — so prefer the qualified anchor whenever it's available (this is the fix for "discoverable but unopenable").

Attempt the read; structural privacy means a genuinely ungranted private doc still returns `PATH_NOT_FOUND` (treat as not-readable — Step 4). Do not open more docs than needed to answer. If a private doc 404s *despite* a present grant and an `item_drive_id`-qualified anchor, that's a cross-drive bug to surface — **do not** fall back to an external connector to fetch it (standards § "Reads go through aifs only").

**Reconcile the pointer against the actual ACL (`libptrstale`).** A pointer's `owner`/`scope` can drift from reality when a doc's ownership or sharing changed **outside the library** — a raw `transferOwnership`, an out-of-band unshare, or an `owner_departed` offboarding. For each candidate you open (or attempt), do a best-effort `aifs_get_permissions` on its `folder_id` and compare to the pointer: if the real owner ≠ the pointer's `owner`, or the grant set no longer matches the pointer's `scope` (e.g. `private_shared` but only the owner still has access, or `private` but others do), **trust the ACL, not the pointer** — surface the corrected owner/scope and note that "the catalog entry for `{title}` is stale (owner/scope changed outside the library)." If you can write the pointer, heal it (rewrite `owner`/`scope`/`item_drive_id` to match the ACL); otherwise flag it for reconciliation (`review-report`'s stale sweep also catches it). Never present a stale pointer's `owner`/`scope` as fact once the ACL disagrees.

### Step 4 — Answer
Summarize the relevant doc(s), cite each by title and group path, and link/point to them. If a strong title match is a private doc the member **can't** read, say so: "There's a private doc titled `{title}` owned by {owner} that looks relevant — you can ask for access with `@ai:request-access`." Never guess its contents.

### Step 5 — Offer next steps
Offer to open a specific doc in full, browse the surrounding group, or request access to a gated match.

---

## Directives

- Metadata-first: route via the catalog; open the minimum set of bodies needed.
- Only read docs the member is authorized to read; never bypass privacy.
- Private, unreadable matches → title-only + request-access; never fabricate or infer content.
- Cite docs by title + group path; prefer the org-public canonical doc when several match.
- Use `aifs_*` reads only; this skill writes nothing.

---

## Error Handling

- Auth/connectivity → halt + `@ai:member-bootstrap`.
- A candidate read returns PATH_NOT_FOUND (private, ungranted) → treat as not-readable; surface title-only; continue with other candidates.
- No matches → say so plainly and offer `@ai:browse-library` to explore the taxonomy.
