---
title: Update Linear team name and workspace slug to Rheged Studio
release_note: ''
created_at: '2026-08-04T18:40:35Z'
merged_at: '2026-08-04T18:52:14Z'
branch: a-1233-shared-workflows-update-linearteamname-linearworkspaceslug
pr: 99
commit: '5886699'
author: rob@acmeskunkworks.io
co_authors: []
category: chore
breaking: false
issues:
  - A-1233
stats:
  files_changed: 36
  loc_added: 88
  loc_removed: 57
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
