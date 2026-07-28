---
title: "Set prefix-development on the npm Dependabot ecosystem"
release_note:
version:
created_at: "2026-07-28T15:13:50Z"
merged_at:
branch: "a-980-dependabot-conventional-commit-templates"
pr:
commit:
author: "rob@acmeskunkworks.io"
co_authors: []
category: chore
breaking: false
issues: ["A-980"]
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Fixed

- The npm ecosystem in `.github/dependabot.yml` set `prefix` but not `prefix-development`
  ([A-980](https://linear.app/acme-skunkworks/issue/A-980)), so only production-dependency
  bumps were prefixed `chore`. Development-dependency bumps fell back to Dependabot's own
  default subject, which is not a Conventional Commit and would fail the commit gate once it
  becomes a required check.

## Notes

- This ecosystem is dev-only by design (its own comment says so), so in practice every bump it
  raises took the unset path — making this the one estate repo where the omission actually bit.
