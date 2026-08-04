---
title: Tighten the GO/NO GO verdict's blanket skipped acceptance
release_note: The canonical GO/NO GO verdict now allowlists skipped only on release-please--* branches, so an all-skipped needs set can no longer mint a green gate with no CI behind it.
created_at: '2026-08-04T09:04:45Z'
merged_at: '2026-08-04T09:18:33Z'
branch: a-1103-tighten-the-gono-go-verdicts-blanket-skipped-acceptance
pr: 98
commit: 99afec0
author: rob@acmeskunkworks.io
co_authors: []
category: fix
breaking: false
issues:
  - A-1103
stats:
  files_changed: 3
  loc_added: 76
  loc_removed: 21
  commits:
---

## Fixed

- **`.github/workflows/ci.yml` ([A-1103](https://linear.app/rheged-studio/issue/A-1103))** —
  the dogfooded `GO/NO GO` verdict now allowlists `skipped` only when
  `github.head_ref` matches `release-please--*`; everywhere else every needed
  job must report `success`. Previously a blanket `success or skipped` accept
  meant an all-skipped `needs` set (stray path filter, misfiring `if:`, scoped
  `edited` re-run) still minted a green gate with no real CI behind it.

## Changed

- **`docs/go-no-go-gate.md`** replaces the blanket-accepting canonical `jq`
  snippet with the same branch-conditional allowlist (octavo's reference shape),
  and rewrites the "never go green off a run that skipped its real CI" footgun
  so that rule is the estate default rather than a deferred follow-up. A repo
  may still tighten further by job name; `HEAD_REF` is passed through `env:`
  alongside `NEEDS_JSON`.
