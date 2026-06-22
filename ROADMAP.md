# Library — Roadmap

## v1.0.0 (this release)

Hierarchical groups (unlimited nesting), documents with images, public/private visibility with member-applied grant lists, an always-public catalog decoupled from content ACLs, governed doc-type vocabulary (admin locked/open), staleness/review tracking (180-day default), and comms-channel access requests.

## Deferred (candidate fast-follows)

- **Private groups / hidden subtrees.** v1 keeps the taxonomy fully org-public; a member who needs privacy uses private *docs* under a public group. A private-subtree feature can be added later without breaking the catalog model.
- **Single-writer / canonical-doc governance.** Public docs are uniform-write with changelog attribution (cooperation, not enforcement). Enforced single-writer ownership is a later option.
- **Tracked request/approve loop.** v1 routes access requests through the org's comms channel; a first-class request queue with status is a possible upgrade.
- **Full-text content search.** v1 retrieval is catalog-metadata-driven (title/tags/summary/type). Indexed full-text search over bodies could be layered on.
- **Cross-doc backlink index.** Stable ids already make links durable; an automatic "what links here" view is a future nicety.
