---
title: Backfill post-merge fields across the changelog history
release_note:
created_at: '2026-07-03T15:55:54Z'
branch: changelog-backfill-post-merge-fields
category: docs
breaking: false
issues:
  - A-597
merged_at: '2026-07-03T16:15:57Z'
commit: 44b089b
merge_strategy: squash
pr: 37
stats:
  loc_added: 157
  loc_removed: 0
  files_changed: 15
---

## Changed

- Backfilled the post-merge frontmatter fields — `branch`, `pr`, `merged_at`,
  `commit`, `merge_strategy`, and `stats` — across all 14 dated entries
  (v0.1.0–v1.1.0), resolving each from its version release tag → merge PR so every
  entry carries its real per-version metadata. Groundwork for [A-597](https://linear.app/acme-skunkworks/issue/A-597) Phase B, so the
  backlog is fully enriched before forward orchestrator-driven enrichment goes live.
