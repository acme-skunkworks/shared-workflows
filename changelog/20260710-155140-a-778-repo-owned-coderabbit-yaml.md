---
title: Document repo-owned .coderabbit.yaml after fan-out removal
release_note: `.coderabbit.yaml` is now documented as repo-owned (A-778 removed the estate fan-out); this repo's copy remains the estate reference profile.
created_at: '2026-07-10T15:51:40Z'
branch: a-778-remove-the-coderabbityaml-fan-out-from-fanout-configyml
author: rob@acmeskunkworks.io
co_authors: []
category: docs
breaking: false
issues:
  - A-778
---

## Changed

- `.coderabbit.yaml` header and `CLAUDE.md` no longer claim a live verbatim fan-out
  from this repo. Consumers own their copies ([A-778](https://linear.app/acme-skunkworks/issue/A-778));
  this file remains the estate reference profile and a strict superset of skip
  conventions for any hand re-sync.
