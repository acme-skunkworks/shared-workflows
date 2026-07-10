---
title: Share load-repo-config via reusable @v1
release_note: Consumers can float reusable-load-repo-config.yml@v1 instead of keeping a duplicated local load-repo-config composite; the action is SHA-pinned inside the reusable.
created_at: '2026-07-10T17:10:58Z'
branch: a-779-move-load-repo-config-into-shared-workflows-reference-at-v1
author: rob@acmeskunkworks.io
co_authors: []
category: feature
breaking: false
issues:
  - A-779
merged_at: '2026-07-10T17:28:07Z'
commit: 8506eca
pr: 70
stats:
  loc_added: 270
  loc_removed: 13
  files_changed: 7
---

## Added

- Layer-1 composite `.github/actions/load-repo-config` — allowlist-validates
  `infrastructure/repo-config.yaml` into step outputs (canonical copy of the
  six-repo duplicate).
- Layer-2 `reusable-load-repo-config.yml` — checkout + SHA-pinned composite;
  exposes all five config knobs as `workflow_call` outputs so callers float
  `@v1` without fighting `sha_pinning_required` on the action itself
  ([A-779](https://linear.app/acme-skunkworks/issue/A-779)).

## Changed

- ADR 0001, `CLAUDE.md`, `README.md`, and the actions catalogue now treat the
  loader as shared via the reusable `@v1` wrapper; `repo-config.yaml` stays
  per-repo.
