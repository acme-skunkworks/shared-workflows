---
title: Allow Cursor bot to trigger Claude Code Review
release_note: Claude Code Review now runs when a Cursor cloud agent opens or updates a PR, by adding cursor[bot] to allowed_bots alongside road-runner-bot[bot].
created_at: "2026-07-23T07:53:45Z"
merged_at:
branch: allow-cursor-in-claude-review
pr:
commit:
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: fix
breaking: false
issues: []
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Fixed

- `claude-code-action` rejects non-human actors unless listed in `allowed_bots`. A
  Cursor cloud agent opens or updates PRs as `cursor[bot]`, so the review job
  failed with "Workflow initiated by non-human actor: cursor". Both
  `reusable-claude-code-review.yml` and the inline dogfood
  `claude-code-review.yml` now allow `cursor[bot]` alongside
  `road-runner-bot[bot]`.
