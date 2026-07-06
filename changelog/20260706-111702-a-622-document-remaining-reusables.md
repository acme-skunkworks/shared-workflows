---
title: Document the lint, build-test and pkg-release reusables in the README
release_note: The README now lists all six reusable workflows and gives copy-ready caller stubs for reusable-lint.yml, reusable-build-test.yml and reusable-pkg-release.yml.
created_at: "2026-07-06T11:17:02Z"
merged_at:
branch: a-622-readme-document-the-remaining-reusables-lint-build-test-pkg
pr:
commit:
merge_strategy:
category: docs
breaking: false
issues:
  - A-622
stats:
  files_changed:
  loc_added:
  loc_removed:
  commits:
---

## Added

- Rows in the README **Available workflows** table for `reusable-lint.yml`,
  `reusable-build-test.yml` and `reusable-pkg-release.yml`, so all six reusable
  workflows now appear (previously only the three Claude / PR-title workflows
  were listed).
- Copy-ready **How to consume** caller stubs for the same three workflows, each
  carrying the top-level `permissions:` block from the _Required caller
  permissions_ table, the canonical caller job id (`lint` / `build-test`, and
  `release` for pkg-release), and a prose line covering the key inputs and lane
  opt-outs. The `pkg-release` stub also documents the branch-protected
  `npm-release` environment prerequisite ([A-326](https://linear.app/acme-skunkworks/issue/A-326)) and the `build: false`
  build-less-package escape hatch. Every stub pins the floating `@v1` tag, in
  step with the rest of the _How to consume_ section.

This closes a pre-existing documentation gap: the three workflows had been added
to the _Required caller permissions_ table in
[A-621](https://linear.app/acme-skunkworks/issue/A-621) but had no intro-table
row and no consume stub, so a consumer of those three got the permissions but no
copy-ready caller ([A-622](https://linear.app/acme-skunkworks/issue/A-622)).
