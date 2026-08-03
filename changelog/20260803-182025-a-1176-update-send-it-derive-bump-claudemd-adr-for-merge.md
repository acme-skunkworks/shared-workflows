---
title: Document dual merge policy for feature vs release/fan-out PRs
release_note: ''
created_at: '2026-08-03T18:20:25Z'
merged_at:
branch: a-1176-update-send-it-derive-bump-claudemd-adr-for-merge-commits
pr:
commit:
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: docs
breaking: false
issues:
  - A-1176
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

**Dual merge policy docs ([A-1176](https://linear.app/acme-skunkworks/issue/A-1176))** — feature / ship PRs use merge commits;
release-please version PRs and fan-out PRs stay squash; both allow-flags remain
on ([A-1177](https://linear.app/acme-skunkworks/issue/A-1177)). Docs-only — no `trunk.json` / allow-flag flip.

- `.github/workflows/reusable-validate-pr-title.yml` — PR title still linted;
  under merge commits it is not the sole bump signal.
- `.github/workflows/reusable-validate-commits.yml` — commitlint is the
  per-commit gate when feature history merges intact.
- `CLAUDE.md`, `README.md`, `SECURITY.md`, `docs/rulesets.md` — replace
  squash-only feature-PR prose with the dual policy; note [A-1177](https://linear.app/acme-skunkworks/issue/A-1177) owns enabling
  merge commits / reconciling `allowed_merge_methods`.
