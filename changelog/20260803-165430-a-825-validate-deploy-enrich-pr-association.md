---
title: Document deploy enrich PR association under merge-commit history
release_note: 'Comments in reusable-changelog-enrich.yml now spell out that commits/{sha}/pulls resolves the associated PR for both squash SHAs (1 parent) and merge commits (2 parents), validated empirically on agent-skills fc7400e / PR #8 (A-825).'
created_at: '2026-08-03T16:54:30Z'
merged_at: '2026-08-03T17:06:56Z'
branch: a-825-validate-changelog-authoring-changelogfinalise-enrichment
pr: 93
commit: 9e49dfa
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: docs
breaking: false
issues:
  - A-825
stats:
  files_changed: 2
  loc_added: 58
  loc_removed: 7
  commits:
---

## Changed

**Document deploy enrich PR association for multi-commit merge history
([A-825](https://linear.app/rheged-studio/issue/A-825))** — the
`reusable-changelog-enrich.yml` resolve step already keyed off
`GET commits/{sha}/pulls` then `gh pr view … mergeCommit` (A-777); comments now
explicitly record that this path works for **both** squash SHAs (1 parent) and
merge commits (2 parents).

Empirical validation (agent-skills historical merge commit `fc7400e`, PR #8):

- **Squash merge** (1 parent): `commits/{sha}/pulls` returns the associated PR;
  `merged_at` is set; prefer `mergeCommit.oid` for the `commit` field, fall back
  to `github.sha` when unset.
- **Merge commit** (2 parents): same association path — PR #8 returned with
  `merged_at` set for `fc7400e`.

No behaviour change — documentation only.
