---
title: Update Linear team name and workspace slug to Rheged Studio
release_note: ''
created_at: '2026-08-04T18:40:35Z'
merged_at: ''
branch: a-1233-shared-workflows-update-linearteamname-linearworkspaceslug
pr:
commit:
author: rob@acmeskunkworks.io
co_authors: []
category: chore
breaking: false
issues:
  - A-1233
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Changed

**Linear identity repair ([A-1233](https://linear.app/rheged-studio/issue/A-1233))**

- Point vendored skill `linearTeamName` / `linearWorkspaceSlug` at Rheged Studio
  (both `.claude/skills` and `.agents/skills` mirrors) so Linear writeback stops
  400ing on the retired ACME Skunkworks team name.
- Rewrite committed `linear.app/acme-skunkworks/…` URLs to `rheged-studio` in
  changelog entries and `docs/security-review-a422.md` (hygiene only — Linear
  still redirects the old workspace slug).
