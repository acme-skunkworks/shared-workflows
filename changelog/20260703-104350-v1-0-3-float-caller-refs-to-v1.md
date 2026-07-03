---
title: "Float consumer caller refs to the @v1 major tag"
release_note: "Consumers now pin their reusable-workflow callers to the floating @v1 tag instead of a per-release SHA — non-breaking releases arrive automatically, with no Dependabot SHA bump."
version: "1.0.3"
created_at: "2026-07-03T10:43:50Z"
category: docs
breaking: false
issues: ["A-662"]
branch: a-662-adopt-floating-caller-tag-pattern-for-the-shared-claude
merged_at: "2026-07-03T11:02:32Z"
commit: 7b11c8c
merge_strategy: squash
pr: 31
stats:
  loc_added: 90
  loc_removed: 22
  files_changed: 6
  commits: 3
---

## Changed

- **Consumer caller stubs now reference the reusable workflows by the floating
  major tag `@v1`** instead of a SHA-pinned caller (`@<sha> # v1.0.2`). New
  non-breaking releases move `v1` forward and land at every consumer's caller
  automatically, removing the per-release Dependabot SHA-bump churn (A-662). This
  is the reusable-workflow **tag exception** to `sha_pinning_required`; true
  `uses:` actions — including `anthropics/claude-code-action` _inside_ the
  reusables — stay SHA-pinned and Dependabot-bumped centrally, so consumers never
  see that pin.
  - Updated the three consumer caller snippets and the Versioning section in
    `README.md`, and the "Consume from another repo" header comments in
    `reusable-claude.yml`, `reusable-claude-code-review.yml` and
    `reusable-validate-pr-title.yml`.
- **Introduced the floating `v1` tag** (tracks the latest `v1.x`) and documented
  the per-release "force-move `v1`" step in `changelog/README.md`. A breaking
  change will ship as a new `v2` major and must not move `v1`. The move is manual
  until A-597 automates the release flow.

The reusable-workflow **job bodies are unchanged** — this release only touches
reference docs and header comments, so `@v1` behaviour is identical to `v1.0.2`.
