---
title: Correct stale internal-pin comment in reusable-pkg-release.yml
release_note:
created_at: '2026-07-20T14:21:58Z'
branch: a-959-repin-pkg-release-setup-project-build
author: rob@acmeskunkworks.io
co_authors: []
category: chore
breaking: false
issues:
  - A-959
merged_at: '2026-07-20T17:22:48Z'
commit: 6488cf0
pr: 81
stats:
  loc_added: 34
  loc_removed: 2
  files_changed: 2
---

## Changed

- The `setup-project` and `build` internal composite-action pins in
  `reusable-pkg-release.yml` were bumped from the pre-release `@d0a5949` SHA to the
  v1.5.1 commit (`950537e`) by the [#80](https://github.com/acme-skunkworks/shared-workflows/pull/80)
  actions-group update, but the trailing `# … @d0a5949 (pre-release internal pin)`
  comment was left behind — actively misleading, since the pin now resolves to a
  released tag rather than a pre-release SHA. Both comments now read `# … @v1.5.1`.
- Comment-only change: the pinned SHAs are unchanged (already at v1.5.1), so there is
  no behavioural change to the release/publish path. Verified that the v1.5.1 action
  content differs from `@d0a5949` only in comment/ticket-reference churn
  ([A-959](https://linear.app/acme-skunkworks/issue/A-959)).
