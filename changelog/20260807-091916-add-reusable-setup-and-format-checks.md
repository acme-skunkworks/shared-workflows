---
title: "Add reusable setup and format checks"
release_note: "Reusable lint and build/test workflows can now run consumer code generation, while lint callers can opt into a Prettier format check."
created_at: "2026-08-07T09:19:16Z"
merged_at:
branch: "a-1280-reusable-lint-reusable-build-test-add-optional-setup-script"
pr:
commit:
author: "rob@acmeskunkworks.io"
co_authors: []
category: feature
breaking: false
issues: ["A-1280", "A-1281"]
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Added

- Added optional post-install `setup-script` inputs to reusable lint and build/test workflows, allowing consumer code generation before checks run ([A-1280](https://linear.app/rheged-studio/issue/A-1280)).
- Added an opt-in Prettier format-check lane to the reusable lint workflow, including consumer guidance for scripts and ignored vendored files ([A-1281](https://linear.app/rheged-studio/issue/A-1281)).
