# Library

The org's documentation library — a Confluence replacement built on agent-index.

Members create **groups** (named categories) and nest **sub-groups** to any depth, then file **documents** (How-Tos, FAQs, Tutorials, Research, Runbooks, Reference, …) under that structure. The taxonomy and every document title are visible to the whole org; document *content* is org-public by default but can be made private at any time.

## Core model

- **Taxonomy is always org-public.** Anyone can see the tree of groups and the titles of the docs in it. (Private *groups* are not supported in v1.)
- **Docs are org-public by default**, and can be set private at creation or any time after.
- **Private = owner + explicit grant list.** A private doc's content lives in the owner's My Drive and is readable only by the owner and members they explicitly grant. There is no owner-bypass.
- **A private doc still shows its title in the public tree.** Only the content (and content-derived metadata) is gated. The system warns you about this every time you make something private — codename sensitive titles.
- **Discovery is catalog-driven.** An always-public catalog (`/shared/library-index/`) holds the tree, a pointer per doc, and the doc-type vocabulary — decoupled from content permissions, so private docs still appear as titles. Rich retrieval metadata (type, tags, summary) is indexed for **public docs only** and is stripped when a doc goes private.
- **Staleness tracking is built in.** Every doc carries an owner, a last-reviewed date, and a review cadence (org default 180 days); `review-report` lists what's overdue.

## How members use it

Reads mostly go through Claude: "what do we have on key rotation?" → `find-doc` matches the catalog, opens the docs you're allowed to read, and summarizes. "Show me the ops section" → `browse-library` renders the subtree (titles only). If you can see a private title but not its content, you can `request-access` and the owner is pinged via the org's configured comms channel.

## Capabilities (13)

Groups: `create-group`, `move-item`, `rename-item`. Docs: `create-doc`, `edit-doc`, `set-doc-visibility`, `archive-doc`. Read: `browse-library`, `find-doc`, `review-report`. Access: `request-access`. Admin: `manage-doc-types`. Onboarding: `library-tutorial`.

## Requirements

agent-index-core 3.9.0+, filesystem adapter 2.5.0+, permission-change-helper 0.4.1+.

See `solution-design-library-collection.md`, `tech-design-library-collection.md`, and `test-plan-library-collection.md` for the full design record.
