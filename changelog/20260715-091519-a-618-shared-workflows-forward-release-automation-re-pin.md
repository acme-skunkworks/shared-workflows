---
title: Re-pin internal action refs to v1.5.0
release_note: ''
created_at: '2026-07-15T09:15:19Z'
branch: a-618-shared-workflows-forward-release-automation-re-pin-internals
author: rob@acmeskunkworks.io
co_authors: []
category: chore
breaking: false
issues:
  - A-618
merged_at: '2026-07-15T09:57:06Z'
commit: '6356491'
pr: 75
stats:
  loc_added: 40
  loc_removed: 12
  files_changed: 4
---

## Changed

- The reusable workflows now pin their sibling composite actions to
  `@ba77002 # v1.5.0` instead of the backfill-era `@7f543be # v0.8.0`, so the
  v1.x workflows stop pinning to an old version of themselves. Affects
  `reusable-lint.yml` (`setup-project`, `eslint`, `lint-markdown`, `lint-yaml`,
  `changelog-validate`) and `reusable-build-test.yml` (`setup-project`, `build`,
  `typecheck`, `test-vitest`, `shellcheck`, `test-bats`)
  ([A-618](https://linear.app/acme-skunkworks/issue/A-618)).
- `reusable-load-repo-config.yml` moves its `load-repo-config` pin off the
  interim [A-779](https://linear.app/acme-skunkworks/issue/A-779) SHA onto the same v1.5.0 commit.

No consumer-facing change: callers still float the reusable workflows at `@v1`,
and the referenced composite-action content is byte-identical between v0.8.0 and
v1.5.0.
