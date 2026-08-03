---
title: Enable merge commits in trunk ruleset and re-vendor send-it
release_note: ''
created_at: '2026-08-03T21:18:00Z'
merged_at: '2026-08-03T20:59:22Z'
branch: a-1177-enable-estate-merge-commits-keep-squash-allowed-for-release
pr: 95
commit: 6003e3d
author: rob@acmeskunkworks.io
co_authors: []
category: chore
breaking: false
issues:
  - A-1177
stats:
  files_changed: 15
  loc_added: 296
  loc_removed: 174
  commits:
---

## Changed

**Estate merge-commit cutover ([A-1177](https://linear.app/acme-skunkworks/issue/A-1177))**

- `.github/rulesets/trunk.json` — `allowed_merge_methods: ["merge","squash"]`
- `docs/rulesets.md` — dual policy landed (no longer deferred to A-1177)
- Re-vendor `send-it` **0.7.0** + refresh `AGENTS.md` (fan-outs paused, A-809)
