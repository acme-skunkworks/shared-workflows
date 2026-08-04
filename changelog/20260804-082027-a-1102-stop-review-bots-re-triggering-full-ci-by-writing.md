---
title: Move CodeRabbit summary into the walkthrough to stop CI thrash
release_note: >-
  CodeRabbit's high-level summary now lands in the walkthrough comment instead of
  the PR description, so review-bot body edits no longer re-fire pull_request.edited
  CI for an unchanged SHA.
created_at: '2026-08-04T08:20:27Z'
merged_at:
branch: a-1102-stop-review-bots-re-triggering-full-ci-by-writing-into-the
pr:
commit:
merge_strategy:
author: rob@acmeskunkworks.io
co_authors: []
category: fix
breaking: false
issues:
  - A-1102
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Fixed

- `.coderabbit.yaml` sets `reviews.high_level_summary_in_walkthrough: true` so the
  high-level summary stays available but no longer edits the PR description
  ([A-1102](https://linear.app/acme-skunkworks/issue/A-1102)). Description edits
  re-fire `pull_request.edited` and waste a full CI run for an unchanged SHA;
  comments do not.

## Changed

- `CLAUDE.md` documents the new key in the estate review-profile section so a
  hand re-sync ([A-778](https://linear.app/acme-skunkworks/issue/A-778)) treats it as intentional policy, not an accidental omission.
